# CountBot 默认上下文注入梳理与可控开关

> 本文回答两个问题：
> 1. **默认情况下，CountBot 每次请求会给大模型注入什么？**（含实测 token 成本）
> 2. **如何逐项关掉这些注入？**（v0.9.x 新增 `injection` 配置 + 设置页「上下文注入」）
>
> 代码基线：HEAD（v0.9.0）。token 为按字符数 ÷ 2 的估算值，实测命令见附录。

---

## 1. 注入链路总览

一次 Web 聊天的完整链路：

```
POST /api/chat/send (backend/api/chat.py)
  ├─ resolve_session_runtime_config()      # 会话级配置覆盖（模型/人设）
  ├─ ConversationContextService.build_model_context()   # 历史 + 会话摘要
  ├─ get_agent_loop()                      # 每请求重建 ContextBuilder / 工具注册表
  │    ├─ ContextBuilder(workspace, memory, skills, persona_config, injection_config)
  │    └─ register_all_tools(..., disabled_tools=config.injection.disabled_tools)
  └─ AgentLoop.process_message()
       ├─ context_builder.build_messages()  ← system prompt + 历史 + 动态 user 上下文
       ├─ provider.chat_stream(messages, tools=registry.get_definitions())
       └─ schedule_context_maintenance()    ← 后台维护（额外 LLM 调用）
```

IM 渠道（`backend/modules/channels/handler.py`）、WebSocket（`backend/app.py` 的 `/ws/chat`）、
cron（`backend/app.py` lifespan）复用同一套共享组件，注入点完全一致。

---

## 2. 默认注入清单（每请求）

### 2.1 System prompt（静态，可被 prompt cache 缓存）

由 `ContextBuilder.build_system_prompt()` 组装（`backend/modules/agent/context.py`），
按顺序拼接以下区块：

| # | 区块 | 来源 | 实测 token | 默认 |
|---|------|------|-----------|------|
| 1 | 身份：名字、运行环境、工作目录、技能目录、临时目录、性格设定（性格保持中文） | `_get_identity()`，性格从 DB `personality` 表读取（失败降级硬编码） | ~120 | 开 |
| 2 | 规则段：CountBot 文档规则、工具与执行（含文件读写约定、wiki）、记忆、安全 | `_get_rules_block()` | ~640 | 开 |
| 2b | 外部编码代理指引：读 `workspace/external_coding_tools.json`，有启用 profile 时展示调用规则 | `_build_external_coding_guidance()` | ~75 | 开 |
| 3 | `always=true` 技能全文（frontmatter `always: true` 的技能整篇注入） | `SkillsLoader.get_always_skills()` + `load_skills_for_context()` | 当前 0（无 always 技能） | 开 |
| 4 | 可用技能摘要：所有已启用技能的 `name + title + description`（内容来自 SKILL.md，语言随技能文件） | `SkillsLoader.build_skills_summary()` | ~780（13 个技能） | 开 |
| 5 | 已激活多智能体团队区块（团队名/模式/成员 + `@团队名` 触发 `workflow_run` 规则） | `_get_active_teams_section()`，查 DB `is_active=True` | 动态（无团队时 0） | 开 |

**默认合计 1,619 tokens**（实测，13 个技能、无激活团队）。英文化前为 **2,150 tokens**。

> 提示词语言：除「性格设定」（用户在设置页维护的中文设定）与技能摘要/SKILL.md 内容
> 之外，所有内置提示词均为英文，详见 §5。
>
> 注：`_get_identity()` 中原本含一段不可达死代码（旧版完整身份模板，`return` 之后的
> ~130 行），本次改造已移除；实际生效内容以表格为准。

### 2.2 工具定义（每轮请求都全量携带）

`ToolRegistry.get_definitions()` 输出全部已注册工具的 JSON Schema。
web-chat 场景默认注册 **13 个工具**（实测）：

| 工具 | token | 说明 | 默认 |
|------|-------|------|------|
| external_coding_agent | ~753 | claude profile 默认启用（`workspace/external_coding_tools.json`） | 开 |
| wiki | ~718 | BM25 知识库，search/get/batch_get/ask/list/stats/create/update/delete/sync | 开 |
| exec | ~650 | Shell 执行 | 开 |
| screenshot | ~479 | 截屏 | 开 |
| workflow_run | ~484 | 多智能体团队（有 subagent_manager 时注册） | 开 |
| web_fetch | ~377 | 网页抓取 | 开 |
| write_file | ~341 | 写文件 | 开 |
| read_file | ~324 | 读文件（兼技能文档读取） | 开 |
| file_search | ~308 | 工作区文件搜索 | 开 |
| edit_file | ~298 | 编辑文件 | 开 |
| memory | ~300 | 统一记忆工具（write/search/read 三合一） | 开 |
| spawn | ~140 | 子代理（有 subagent_manager 时注册） | 开 |
| send_media | ~194 | 发媒体（有 channel_manager 时注册） | 开 |
| list_dir | ~123 | 列目录 | 开 |

**默认合计约 4,600 tokens**。MCP 工具（`mcp_*` 前缀）仅在 `mcp.enabled=true`
且配置了服务器时追加；小智 `send_message` 仅在小智渠道对话模式开启时注册。

### 2.3 首条 user 消息的动态注入

`build_messages()` 把随会话变化的内容拼到**首条 user 消息**（刻意不放在 system，
避免破坏 prompt cache 前缀稳定性，见 `tests/test_prompt_cache_stability.py`）：

| 内容 | token | 默认 |
|------|-------|------|
| 当前时间（精确到分钟 + 星期） | ~30 | 开 |
| 用户称呼 / 常用地址 / 默认输出语言 | ~40 | 开 |
| 会话摘要 `session.summary`（前端可见的会话级摘要） | 动态 | 开 |
| 渠道信息 Channel / Chat ID / Account ID | ~30 | 开 |
| 附件路径提示（本轮消息的工作区附件清单） | 动态 | 开 |
| `@团队名` 检测 → 大段 `workflow_run` 强制调用指令（含最近 5 条对话上下文） | 500–1500 | 开 |

### 2.4 对话历史

- `persona.max_history_messages` 默认 **0 = 不限**，全量历史进入请求（仅受模型上下文窗口约束）。
- 历史中 assistant 消息的 `reasoning_content` 会从 `message_context` 提取并一并注入。
- `persona.enable_short_context_summary` 默认 **关闭**；开启后超过窗口的历史被
  「短期摘要 system 消息 + 尾部原始消息」替代。

### 2.5 后台 LLM 调用（不注入请求，但消耗 token 配额）

由 `schedule_context_maintenance()`（`backend/modules/session/context_service.py`）
在每轮回复后异步执行：

| 任务 | 触发条件 | 默认 |
|------|---------|------|
| 短期上下文摘要缓存刷新 | `enable_short_context_summary=true` 且超窗 | 关（前置开关关） |
| 溢出历史 → MEMORY.md 总结 | `max_history_messages>0` 且溢出 ≥3 条 | 受限于前置开关 |
| 整会话自动沉淀 → MEMORY.md | 会话 ≥30 条消息或 ≥15000 字符 | **开** |
| 心跳主动问候 | `persona.heartbeat.enabled` | 关 |
| 渠道历史压缩（>15 条时 summarize 旧历史为 system 摘要） | 仅 IM 渠道 | 开（渠道链路固有） |

---

## 3. 可控开关（新增）

### 3.1 配置模型

`backend/modules/config/schema.py` 新增 `InjectionConfig`，挂载为 `AppConfig.injection`，
持久化在 DB `config.injection.*` 键下（旧配置自动获得默认值，无需迁移）：

| 字段 | 默认 | 关闭效果 | 预计节省 |
|------|------|---------|---------|
| `core_rules` | true | 规则段整段消失，仅留最小身份（名字/环境/目录/性格） | ~750 |
| `external_coding_guidance` | true | 规则段中不再出现外部编码代理指引 | ~60 |
| `skills_always` | true | `always=true` 技能不再全文注入 | 动态 |
| `skills_summary` | true | 模型看不到技能列表（无法按需触发技能） | ~975 |
| `agent_teams` | true | 不注入激活团队区块（`@团队` 直连路径仍可用） | 动态 |
| `dynamic_context` | true | 首 5 项动态字段（时间/用户资料/输出语言/会话摘要/渠道）完全不发送，模型不知道当前时间与用户称呼。注意：身份（名字/性格）与运行环境不受此开关控制，模型仍可能从工作目录路径推断系统用户名 | ~150 |
| `team_reminder` | true | `@团队名` 不再触发大段强制调用指令 | 500–1500 |
| `disabled_tools` | [] | 列出禁用的工具名，不出现在工具定义里 | 按工具 |
| `auto_memory_summary` | true | 跳过溢出/整会话两类后台记忆 LLM 调用 | 后台配额 |

### 3.2 生效范围（保存即生效）

保存设置后**不需要重启、不需要重连**，三条链路都已覆盖：

| 链路 | 生效方式 |
|------|---------|
| Web 新消息（SSE `/api/chat/send`） | 每请求重读 `config_loader.config` 重建 ContextBuilder + 工具注册表，天然即时 |
| Web UI 长连接（WebSocket `/ws/chat`） | ContextBuilder 是共享对象，下一条消息即用新开关；工具集由 `sync_disabled_tools()` 在每条消息前对齐（同 MCP 工具的 reconcile 钩子） |
| IM 渠道（微信/飞书/钉钉/QQ/企微/微博/Telegram/小智） | `PUT /api/settings` → `_reload_injection_runtime()` → `handler.reload_config(injection_config, tool_params_updates)` 并重建注册表 |
| cron 定时任务 | 同上，重建 `cron_agent.tools`；其 ContextBuilder 与共享对象同一个 |
| spawn 子代理 | 子代理直接读 `config_loader.config.injection`，其技能摘要与工具集跟随开关 |

`_reload_injection_runtime()` 是唯一的热重载入口，避免在多处重复拼装。

### 3.3 注意事项

- 关 `skills_summary` 后模型不知道有哪些技能，技能类请求基本不可用——面向
  「只要一个干净聊天机器人」的场景。
- 禁用核心工具（`read_file` / `write_file` / `edit_file` / `exec`）会破坏绝大多数
  技能的读取与执行链路，设置页有明确警示。
- `memory` 工具禁用后，手动记忆读写不可用，但 `auto_memory_summary` 关闭只是停掉
  后台自动沉淀，两者独立。
- `dynamic_context` 关闭后模型不知道当前时间与用户称呼，依赖时间的技能（新闻、
  日程）会退化。

---

## 4. prompt cache 视角：哪些注入会打断缓存

缓存命中的前提是**前缀字节级稳定**。按注入位置逐项核对：

### 4.1 已处理：system prompt 内的不稳定与重复

| 问题 | 影响 | 处理 |
|------|------|------|
| 团队区块查询缺 `ORDER BY`，DB 返回顺序不保证 | 每请求 system prompt 可能重排 → 前缀整体失效 | `_get_active_teams_section` / `_get_active_team_names` 增加 `order_by(AgentTeam.name)` |
| 「技能文档读取方式」在 `## 文件与技能` 与技能区引导语重复 | 每请求白付 ~40 tokens | 删除规则段中的重复行，技能区引导语压缩为一句 |
| 技能区引导语 4 句长文案 | 每请求固定开销 | 压缩为「技能是文档不是工具：用 read_file 读 SKILL.md…」 |

### 4.2 已确认稳定（无需处理）

| 注入内容 | 为什么稳定 |
|---------|-----------|
| 名字 / 运行环境 / 工作目录 | 进程级常量，仅改工作区配置时变 |
| 性格设定 | 配置驱动，仅用户在设置页修改时变 |
| `always` 技能全文 / 技能摘要 | 按技能名排序输出；`SkillsLoader` 内存缓存，仅技能增删/开关时变 |
| 外部编码代理指引 | 读 `external_coding_tools.json`，仅配置变更时变 |
| 工具定义 | `get_definitions()` 对内置/MCP 分组后按名称排序，顺序稳定；工具集合只在配置变更时变 |
| 当前时间 / 用户资料 / 会话摘要 / 渠道信息 / 团队提醒 | 刻意注入到**当前轮 user 消息尾部**，不进入 system；不影响历史前缀 |

### 4.3 仍会影响缓存的动态内容（按影响从大到小）

| 来源 | 触发 | 影响 | 建议 |
|------|------|------|------|
| **短期上下文摘要**（`persona.enable_short_context_summary`） | 超窗时刷新 | 在历史头部插入一条 **system** 消息（`## Previous Conversation Summary`），其后全部历史失效 | 默认关闭，缓存敏感场景保持关闭 |
| **IM 渠道历史压缩**（`handler` 内 `>15` 条触发） | 每轮检查 | 用一条摘要替换早期历史 → 当轮前缀全变 | 渠道链路固有行为；高频长会话可调大触发阈值 |
| **MCP 工具动态接入/断开** | 连接状态变化 | 工具定义集合变化 → 请求前缀（tools 字段）变化 | 已按名称排序，集合稳定时不失效 |
| **技能增删/启停**（含 `find-skills` 安装新技能） | 用户操作 | 技能摘要整体变化 | 属预期配置变更；避免会话中频繁装卸技能 |
| **人格/团队/外部编码配置编辑** | 用户操作 | 对应区块变化 | 预期行为 |

> 结论：**默认配置下，system prompt 与工具定义在会话内是字节稳定的**；
> 唯一的内建动态项是「短期上下文摘要」（默认关）和「渠道历史压缩」（IM 链路）。

---

## 5. 提示词语言：英文 + 实测收益

**原则**：内置提示词（system prompt、子代理 prompt、摘要/问候/定时任务模板）统一英文，
因为同等信息量下英文 token 更少（cl100k_base 实测：中文约 1 token/字，英文约 4–5 字符/token）。
**例外**：性格设定（用户在设置页维护的中文文案）、技能摘要与 `SKILL.md` 内容、
团队名/成员/用户数据、日志与注释——保持原语言。

输出语言由 persona 的 `output_language` 控制（动态上下文里传给模型），
所有产出用户可见文本的 prompt 都明确要求「write in the user's language」，
因此英文化不会让中文用户收到英文回复。

实测（tiktoken `cl100k_base`，对照 HEAD 版本）：

| 注入块 | 改前 | 改后 | 节省 |
|--------|-----:|-----:|-----:|
| system prompt（每请求） | 2,150 | 1,619 | **-531（25%）** |
| `@团队` 提醒（每次 @ 触发） | 683 | 183 | -500（73%） |
| 附件提示（带附件时） | 146 | 81 | -65（45%） |
| 动态 user 上下文 | 86 | 74 | -12（14%） |
| 子代理 prompt | 266 | 175 | -91（34%） |
| 记忆总结 prompt | 258 | 148 | -110（43%） |
| 短期摘要 prompt | 277 | 129 | -148（53%） |
| 溢出总结 prompt | 282 | 154 | -128（45%） |
| 心跳问候 prompt | 265 | 189 | -76（29%） |
| 定时任务 prompt | 209 | 133 | -76（36%） |
| **合计** | **4,622** | **2,885** | **-1,737（38%）** |

换算到实际调用：**每条普通消息省 ~531 tokens**；带 `@团队` 的消息再省 ~500；
后台记忆维护每次省 ~110–148。

配套改动：
- `prompts.py` 增加 `is_memory_noop()`，统一识别 `NOTHING_TO_RECORD`（兼容旧的「无需记录」）。
- 子代理的 reasoning 分段标题改为 `## Reasoning` / `## Response`，前端
  `reasoningSections.ts` 已同时支持中英标题。
- 动态上下文的三个近似标题收敛为 `## Session Context` / `## Session Summary` / `## Channel`。

---

## 6. 相关代码位置

| 功能 | 文件 |
|------|------|
| 注入配置模型 | `backend/modules/config/schema.py` → `InjectionConfig` |
| system prompt 组装与开关 | `backend/modules/agent/context.py` → `ContextBuilder` |
| 内置提示词模板（英文） | `backend/modules/agent/prompts.py`（含 `is_memory_noop`） |
| 工具注册 / 禁用 / 长连接热同步 | `backend/modules/tools/setup.py` → `register_all_tools` / `sync_disabled_tools` |
| 后台记忆维护开关 | `backend/modules/session/context_service.py` → `schedule_context_maintenance` |
| 启动接线（含 cron/ws） | `backend/app.py` → `_create_shared_components` / lifespan |
| Web 聊天接线 | `backend/api/chat.py` → `get_agent_loop` / `send_message` |
| WebSocket 每消息对齐工具集 | `backend/ws/events.py` |
| IM 渠道接线 | `backend/modules/channels/handler.py` → `reload_config` |
| 设置 API（GET/PUT + 唯一热重载入口） | `backend/api/settings.py` → `_reload_injection_runtime` |
| 子代理一致性 | `backend/modules/agent/subagent.py` |
| 设置页 | `frontend/src/modules/settings/InjectionConfig.vue` + `styles/InjectionConfig.css` |
| 回归测试 | `tests/test_injection_switches.py`（18 例） |

### 新增：请求预览（排查注入问题）

「开关关了为什么还有」「模型怎么知道我的名字」这类问题，逐项猜代码效率很低。
现在可以直接查看一次请求的真实内容：

```bash
POST /api/chat/preview
{"session_id": "<会话ID>", "message": "拟发送的内容"}
```

返回内容包含：

| 字段 | 说明 |
|------|------|
| `sections` | system prompt 的每个区块：名称、字符数、**受哪个注入开关控制**、当前是否启用 |
| `system_prompt` | 完整拼装结果，与真实请求一致 |
| `first_user_message` | 首条用户消息全文（含追加的动态上下文） |
| `dynamic_context` | 从首条用户消息中分离出的动态上下文部分，关闭时为空 |
| `tools` / `tool_chars` | 本次会携带的工具定义清单与体积 |
| `injection` | 当前所有注入开关的实际取值 |

实现位于 `backend/modules/agent/request_preview.py`，`build_system_prompt_sections()`
按区块返回内容，拼装结果与 `build_system_prompt()` 完全一致（有回归测试保证）。

### 新增：会话并发观测

同一会话现在严格串行处理，排队情况可通过以下接口查看：

```bash
GET /api/system/concurrency
```

返回正在处理的会话、已运行秒数、排队消息数。若某个会话长时间处于 busy，
说明上一轮仍在执行（可能卡在长任务工具里），新消息正在排队等待。

### 新增：模型用量记录

每次模型调用都会记录一条用量（表 `model_usage`），包含输入/输出/缓存读写/推理
token。查询方式：

```bash
GET /api/system/usage?days=7            # 全库
GET /api/system/usage?session_id=xxx    # 单会话
```

返回总量、按用途（对话/摘要等）拆分、按模型拆分，以及缓存命中率——可以直接用来
验证注入开关到底省了多少 token。


---

## 附录：实测方法

```bash
# 工具定义与 system prompt 的 token 实测
cd CountBot && .venv/bin/python - <<'EOF'
import json
from pathlib import Path
from backend.modules.tools.setup import register_all_tools
from backend.modules.agent.skills import SkillsLoader
from backend.modules.agent.context import ContextBuilder

ws = Path("workspace")
reg = register_all_tools(workspace=ws, memory_store=None, subagent_manager=None)
total = sum(len(json.dumps(d, ensure_ascii=False)) for d in reg.get_definitions())
print("tools:", len(reg.get_definitions()), total, "chars ≈", total // 2, "tokens")

skills = SkillsLoader(ws / "skills")
print("skills summary:", len(skills.build_skills_summary()) // 2, "tokens")
cb = ContextBuilder(workspace=ws, memory=None, skills=skills, persona_config=None)
print("system prompt:", len(cb.build_system_prompt()) // 2, "tokens")
EOF
```

> 沙箱注意：本机 WorkBuddy 沙箱会对已存在路径的 `mkdir(exist_ok=True)` 误报
> `PermissionError: EEXIST`，这是环境限制而非代码缺陷；跑 pytest 时用
> `--basetemp=<新目录>` 规避，6 个 collection 阶段因此失败的文件与本次改动无关
> （已用 `git stash` 对照验证）。
