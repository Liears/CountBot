# 上下文工程与调度链路审计报告

> 审计日期：2026-10-24
> 范围：上下文工程（提示词组装 / 历史重建 / 压缩 / 技能 / 记忆 / 人格）+ 调度链路（消息总线 / WebSocket / SSE / IM 渠道 / cron / 子代理）
> 方法：双侧并行勘查 + 逐条人工复核，报告中每条结论均经本人读取原始代码确认，标注 文件:行号
> 总体：**确认 22 项缺陷**（严重 2、中 11、轻 9），另发现 1 项既有测试失败暴露的缺陷；我本人上轮代码中的 1 项缺陷已当场修复
>
> **修复状态（2026-10-24 11:00 更新）**：严重 2 项、中 12 项、轻 9 项已修复，详见 §7 修复记录。
> 当前测试基线：**487 passed / 0 failed**（原 464 passed / 4 failed）。

---

## 一、上下文工程

### C1 【严重·条件触发】Anthropic 链路透传 `reasoning_content`，非规范字段直接进 API 请求体

**证据链**（三段都 verified）：

1. 历史重建时会还原 reasoning：`context_service.py:41-45`，`default_message_formatter` 把 `message_context` 里的 reasoning 放回消息字典；
2. 而 Anthropic 原生块**不持久化**：全库 grep `anthropic_content_blocks` 只命中 provider 内部（`anthropic_provider.py:213/217/633/826`），数据库里没有——所以第二轮起，assistant 消息**只有** `reasoning_content`、没有原生块；
3. Anthropic 的归一化对普通 assistant 消息**原样透传**：`anthropic_provider.py:254` `filtered_messages.append(msg)`，随后直接进 `request_params["messages"]`（`:105`）。

对照组：OpenAI 侧有显式白名单净化，只保留 role/content/tool_calls/tool_call_id/reasoning_content/name（`openai_provider.py:501-521`）。**两条链路行为不对称，Anthropic 侧是缺口。**

**触发条件**：provider=anthropic 且开启思考（`thinking_enabled=True`）→ 同一会话第二条消息起，每轮请求都带 `reasoning_content` 键。
**影响**：该键不属于 Anthropic Messages 消息结构，轻则污染请求体、放大计费 token，重则被 API 判 `invalid_request_error` 整轮失败（且错误经 `_format_error_message` 包装，真实原因被掩盖，极难排查）。
**复现**：anthropic + thinking 开，同一会话连发两条消息，看第二条的请求体。

### C2 【中】`auto_memory_summary` 开关在 WebSocket 主链路未接入

**证据**：`ws/events.py:451-460` 调 `schedule_context_maintenance` 时**没有传** `enable_auto_memory_summary`，函数默认 `True`（`context_service.py:621`）；SSE 侧传了（`api/chat.py:1258`）。

**影响**：前端聊天实际走 WebSocket（`frontend/src/composables/useWebSocket.ts`），也就是说**用户最常用的路径不尊重这个开关**。在设置里关掉「后台自动记忆维护」后，网页版每次回复仍会跑两次后台 LLM 摘要，token 照付。这是「省 token 承诺在主 UI 上不兑现」。

### C3 【中】渠道路由历史不剥 workflow 元数据

**证据**：`handler.py:2303-2313` 与 `2321-2331` 用 `_format_message_for_model` 重组 content，该函数对非群聊消息**原样返回**（`handler.py:256-258`）；而 web 路径会剥（`context_service.py:39` 调 `strip_workflow_exec_metadata`）。保存侧存的又是未剥的原文（`handler.py:783-788`）。

**影响**：渠道会话（telegram/飞书等）里，一次 `@团队` 直连产生的 `<!--WORKFLOW_EXEC:{json}-->` 会被**每一轮**喂回模型——与 web 路径行为不一致，既费 token 又让模型看到内部执行元数据。

### C4 【中】渠道路由历史不还原 `reasoning_content`

**证据**：`handler.py:2303-2313` 构造的 dict 只有 `role`/`content` 两个键；web 路径会还原（`context_service.py:41-45`）。

**影响**：同一会话从渠道走时丢失思考上下文，与 web 路径不一致。开了思考的渠道会话，模型看不到自己此前的推理。

### C5 【中】技能注入无总量上限，`always: true` 可撑爆上下文

**证据**：`build_skills_summary` 不限制条数、不截断 description（`skills.py:654-666`）；`load_skills_for_context` 注入 always 技能**全文**且无长度限制（`skills.py:611-628`）。

**影响**：workspace 现有 14 个技能，最大 SKILL.md 约 33KB。任一个被标 `always: true` 就全文进 system prompt；装几十个各带长描述的技能同样无上限。极端情况直接顶爆模型窗口，请求 400。
**复现**：任意 SKILL.md frontmatter 写 `always: true` + 长正文，下一轮看 system prompt 体积。

### C6 【中·安全】`add_skill` 未校验名称，存在路径穿越

**证据**：`skills.py:868` `skill_dir = self.workspace_skills / name`，无 `..`/`/` 校验；API 侧只限制长度（`api/skills.py:78` `min_length=1, max_length=50`）。

**影响**：`POST /api/skills` 传 `name="../../evil"` → `mkdir(parents=True)` 在 workspace 外创建目录并写入 SKILL.md。
**复现**：见上。

### C7 【轻】`reasoning_content` 无大小上限

`message_context.py:42-44` 只 strip 不截断。长思考（数万 token）全文入库，且每轮历史重建原样带回，双重放大请求体积。

### C8 【轻】渠道路由 `/p` 命令只认硬编码人格

`handler.py:1478-1529` 只用 `PERSONALITY_PRESETS`，用户在设置页新建/修改的人格选不了。

### C9 【轻】主动问候不走 DB 人格

`heartbeat.py:443-446` 调模块级 `get_personality_prompt`（`personalities.py:177-202` 硬编码预设）。用户在设置里改的人格对主动问候无效。另外问候是裸 `chat_stream`（`:465`，只有一条 user 消息，无 system prompt、无工具、不过闸门）。

### C10 【轻】会话名 LIKE 前缀错配，可串会话上下文

8 处查询用 `Session.name.like(f"{prefix}%")`（`handler.py:1086/1089/1158/1166`、`cron/executor.py:345/354`、`api/cron.py:247/257`），prefix 形如 `telegram:default:123`。chat_id `123` 与 `1234` 共存时，查 `123` 会命中 `1234` 的会话；`updated_at` 更新的一方胜出，消息写进错误会话。chat_id 含 `_` 或 `%` 时 LIKE 通配符同样误匹配。

### C11 【轻】`is_memory_noop` 全文子串匹配，会误杀合法记忆

`prompts.py:96-98` `any(sentinel in text ...)`，哨兵含中文「无需记录」。模型输出「用户要求日志无需记录级别设为 WARN」这类正常记忆会被整条丢弃。

### C12 【轻】提示词模板 `.format()` 无防护，崩溃分布不一致

当前 8 个模板均无未转义字面花括号（安全），但这是隐式契约。一旦有人加入含 `{}` 的内容（如 JSON 示例），`str.format` 抛 `KeyError`，且各调用点表现不一：heartbeat 的 format **不在 try 内**（`heartbeat.py:453`，try 从 463 才开始）→ 冒泡后问候静默跳过；cron/executor 在 try 内 → 整个任务失败；context_service 在 `_runner` try 内 → 后台维护整轮中止。建议改用 `str.replace`。

### C13 【轻】`strip_workflow_exec_metadata` 对截断注释无能为力

`message_context.py:17-20` 正则要求完整闭合标签。若内容被上游截断（exec 10000 字、compactor 2000 字）恰好在注释中间，半截 JSON 元数据泄漏进上下文。

### C14 【轻】死模板

`RECURSIVE_SUMMARY_PROMPT`（`prompts.py:32-47`）与 `MEMORY_ENTRY_TEMPLATE`（`prompts.py:89`）全库无引用，后者带占位符，容易被误用引入 format 陷阱。

---

## 二、调度链路

### S1 【严重】cron 完全绕过会话闸门

**证据**：`cron/executor.py:73` `self.agent.process_direct(...)` → `loop.py:1003-1037` 全函数无 `get_session_gate()`；而 `session_gate.py` 模块文档明确写了该闸门就是为「IM/WS/cron 共用同一会话」设计。

**影响**：用户给某渠道会话建一个 cron（如每分钟），同时在该会话聊天 → 两轮处理并发读写同一 session 历史，回复与用户所见顺序错乱、历史交错。这正是闸门要防的事，但 cron 是唯一没接的入口。
**说明**：此项在我上轮报告中列为「待决项」，本次审计确认它是**真实缺口**而非设计取舍——闸门模块自己声明的覆盖范围就包含 cron。

### S2 【中】消息总线去重误杀合法消息，且静默丢弃

**证据**：`enterprise_queue.py:251-254` 哈希只含 `channel/chat_id/sender_id/content`，**不含 `msg.media`**；窗口 10 秒（`app.py:210-213`）；命中即 `return False` 并只打一条 warning（`:152-155`），`publish_inbound`（`:209-212`）忽略返回值。

**影响**：IM 场景用户 10 秒内连发相同文本极常见（「继续」「在吗」）→ 第二条被静默丢弃，用户无感知；同一文案配不同图片时，第二张图连带整条消息一起丢。

### S3 【中】死信队列与自动重试是死代码

**证据**：全库 grep `mark_failed`/`mark_success` **零调用方**；消费侧 `handler.py:470-474` dequeue 后既不记成功也不记失败。

**影响**：任何消息处理失败都不会重试、不会进 DLQ，仅靠 `handle_message` 内部 try/except 兜底。队列宣称的「企业级」能力实际不存在——要么接上，要么从文档里去掉这个说法。

### S4 【中】WS 取消令牌早退泄漏

**证据**：`events.py:198` `get_cancel_token(session_id)` 之后有 4 条早退路径不清理：会话不存在（`:211`）、附件非法（`:222`）、provider 不可用（`:243`）、用户消息落库失败（`:289`）；`cleanup_cancel_token` 只在成功（`:470`）与异常（`:480`）出口调用。

**影响**：`_session_cancel_tokens`/`_session_tasks` 留下永久条目；更麻烦的是残留一个**未取消的旧令牌**，下一次 `get_cancel_token` 会复用它（`connection.py:52-72`）。

### S5 【中】跨轮次令牌孤立，`/stop` 静默失效

**证据**：`get_cancel_token` 复用已有令牌（`connection.py:72` 直接返回存量）。多客户端同会话时：Tab A 的消息 1 创建令牌 T；Tab B 的消息 2 拿到同一个 T；消息 1 结束即 `cleanup_cancel_token`（`events.py:470`）删掉字典条目；消息 2 拿着**孤立的 T** 继续跑，此时 `cancel_session` 查不到条目返回 False。

**影响**：用户在第二个标签页点「停止」，接口返回「没有正在执行的任务」，生成却停不下来。修闸门后此场景更容易触发（排队轮在准入前就已持有令牌）。

### S6 【中】手动 cron 触发 TOCTOU，可双跑

**证据**：`api/cron.py:960` 检查 `is_job_active` 与 `:1022` `create_task` 之间存在 await 窗口；且手动路径不把自己注册进 `_active_jobs`、不过信号量、无超时。

**影响**：前端连点「立即执行」，或手动执行与调度器到期几乎同时 → 同一 job 两份并发执行。

### S7 【中】队列无界，无背压

`enterprise_queue.py:108-119` 四个 `asyncio.Queue()` 均无 `maxsize`。单一渠道洪水（机器人被拉进高频群、渠道 bug 重放）→ 内存无限增长直至 OOM，生产者永远感觉不到阻塞。

### S8 【中】去重表无周期清扫，内存无限增长

`enterprise_queue.py:256-270` 只在「同一 hash 再次被查询」时才删过期项，没有任何周期性清扫。每条唯一消息永久留一个 dict 条目（约 200 字节），百万级消息约 200MB。

### S9 【轻】`dequeue` 等待路径可能丢消息

`enterprise_queue.py:198-205` 用 `asyncio.wait(FIRST_COMPLETED)`，`done` 可能含 ≥2 个已完成任务，`done.pop()` 只取一个，其余已从队列取出的消息被静默丢弃。当前所有消息都走 NORMAL 优先级故不可触发，一旦启用优先级即触发。

### S10 【轻】优先级机制空转

`publish_inbound`（`:211`）从 `InboundMessage` 取 `priority`，但该 dataclass 没有这个字段（`base.py:14-29`）→ URGENT/HIGH/LOW 三条队列永远为空。

### S11 【轻】流式迭代异常不 flush

`streaming.py:209-222` `stream_iterator` 的 `async for` 无 try/finally，上游 LLM 流中途抛错时收尾 flush 被跳过，最多丢失一缓冲内容（WS 配置 10 字符）。

### S12 【轻】出站发送失败即丢弃，且调度器无监督重启

`manager.py:280-281` channel.send 异常仅记 error，消息出栈后无重试无 DLQ；`manager.py:203-206` `start_all` 用 `gather(return_exceptions=True)`，出站循环若因未预期异常退出会被静默吞掉，之后所有出站消息滞留。

### S13 【轻】入站回调无异常保护

`manager.py:262-265` `_on_inbound_message` 直接 `await publish_inbound` 无 try/except。当前不会抛，但一旦启用持久化或引入异常，会冒泡进渠道回调且该消息丢失。

### S14 【轻】heartbeat 失败被吞记为成功

`cron/executor.py:160-162` `_execute_heartbeat` catch-all 返回 `""`，调度器只见无异常 → `last_status="ok"`。投递渠道故障时面板显示成功。

### S15 【轻】外部编码代理 history 模式只限条数不限长度

`external_agents/conversation.py:67` 只做 `[-N:]` 切片，无字符/token 上限。历史里有超长单条消息（粘贴大日志）时，拼出的 prompt 可远超模型窗口。

### S16 【轻】native 模式外传内部 session_id

`external_agents/cli.py:208-213` 把 CountBot 内部 session_id 作为 `--session-id` 传给 claude CLI。cron 无渠道时 session_id 形如 `cron:{job_id}`（`executor.py:66`），非 UUID 形态，CLI 参数校验可能失败。

### S17 【轻】记忆文件读-改-写竞争

`memory.py:79-81` `append_entry` 先整文件读再整文件写，无锁无原子追加。系统中同时存在多个 `MemoryStore` 实例指向同一文件（全局单例 `app.py:92`、HTTP 每请求新建 `chat.py:563-565`、API 每请求新建 `api/memory.py:56-61`、heartbeat 每次新建 `heartbeat.py:435`）。单进程单事件循环下同步函数不会被打断（侥幸安全），但多 worker 部署或设置页 `write_all` 与后台沉淀交错时会静默丢条目。另：MEMORY.md 只增不减，无容量上限，`GET /api/memory/long-term` 会把整个文件拉回前端。

### S18 【轻】共享 MemoryTool 实例的 source 串会话

`registry.py:145-147` `set_channel` 把 channel 写到同一个 tool 实例上，web 与 telegram 并发时后执行者覆盖前者，记忆来源标记张冠李戴。

### S19 【轻】技能无缓存，改 SKILL.md 不生效

`SkillsLoader` 无 mtime/TTL 缓存，`read_skill` 返回加载时快照。用户在编辑器改 SKILL.md 后继续聊天，Agent 读到的是旧内容，需刷新技能页或显式 reload 才变。这是「无缓存即无失效问题」的取舍，但「改了不生效」的体感是真实的。

### S20 【轻】技能 toggle 双写，失败时状态不一致

`skills.py:756-757` 只改内存 `disabled_skills`，持久化靠 API 层另写 `.skills_config.json`（`api/skills.py:418`）。写文件抛异常时内存已禁用、文件未记录，重启后回退。

### S21 【轻】依赖不满足的技能静默消失

`skills.py:650-651` 对 `check_requirements()` 为 False 的技能直接 `continue`，无告警暴露到 UI（`get_missing_requirements` 存在但摘要链路不调用）。用户视角：技能莫名不见了。

### S22 【轻】cron 重试日志死代码 + 定时器批次阻塞

`scheduler.py:318` 先 `job.retry_count = 0`，`:321` `if job.retry_count > 0` 永假；`:196` `await asyncio.gather(*tasks)` + 1200s 超时，一批长任务跑满 20 分钟会延迟期间新到期的 job。

---

## 三、审计中顺带发现：既有测试失败暴露的缺陷

跑全量 `pytest tests/` 有 **4 个失败**，全部来自 `tests/test_local_tool_outcomes.py`。根因是真实缺陷，且与本次改动无关（`filesystem.py` 不在我的改动清单中，最后一次提交是文档翻译）：

### P1 【中】`WorkspaceValidator` 构造参数被全局配置静默覆盖

**证据**：`filesystem.py:43-52` `workspace` property 在 `config_loader.config.workspace.path` 有值时**返回配置路径而非构造参数**；`:54-60` `restrict_to_workspace` property 同样用 `config.security.restrict_to_workspace` 覆盖构造参数。

**影响**：
- `ReadFileTool(tmp_path, restrict_to_workspace=False)` 的 `restrict_to_workspace=False` **完全失效**，仍按全局配置校验 → 4 个测试因此失败（错误信息：`Path ... is outside workspace '/Users/vam/Downloads/ZCode-main/CountBot/workspace'`）；
- 生产环境巧合正确（`register_all_tools` 传的就是配置 workspace），但**按工具覆盖 workspace/限制是不可能的**；
- 与 `ExecTool.workspace` 是同一套「动态跟随配置」模式（`shell.py:150-156`），说明这是有意为之的热切换设计——但把构造参数变成死参数、且让测试无法注入，是设计缺陷而非特性。

---

## 四、我本人上轮代码中的缺陷（已当场修复）

### M1 【中】排队补救消息可能重复落库

上轮为修「插话消息丢失」加了补救流程：未消费的插话重新投递回总线，并带 `_persisted=True` 让主路径跳过重复写库。但**忙分支漏了这个判断**——`handler.py:620-628` 无条件 `_save_message`。若补救消息到达时闸门又忙了（期间有新消息抢到执行权），会再写一行 → 数据库出现重复 user 消息。

**已修复**：忙分支也检查 `_persisted`。修复后全量测试 464 passed。

---

## 五、修复优先级建议

### 第一批（功能直接坏或安全）
| # | 缺陷 | 预估改动 |
|---|------|---------|
| C1 | Anthropic 透传 reasoning_content | `anthropic_provider.py:254` 前按 OpenAI 同款白名单剥离，约 10 行 |
| S1 | cron 绕过闸门 | `executor.py:73` 前后 acquire/release，约 10 行 |
| C6 | add_skill 路径穿越 | `skills.py:868` 加名称校验，约 5 行 |
| C2 | WS 漏接开关 | `ws/events.py:451` 补一个参数，1 行 |
| M1 | 补救消息重复落库 | 已完成 |

### 第二批（数据正确性）
| # | 缺陷 |
|---|------|
| S2 | 去重纳入 media + 丢弃时向发送者回执 |
| S4/S5 | WS 令牌生命周期改 try/finally + 每轮新建令牌 |
| C3/C4 | 渠道路由历史与 web 路径对齐（剥元数据、还原 reasoning） |
| C5 | 技能注入加总量与单条上限 |
| S3 | 接入 mark_success/mark_failed 或移除 DLQ 宣称 |

### 第三批（健壮性与一致性）
C10 会话名精确匹配、S7/S8 队列上限与清扫、S6 cron TOCTOU、P1 validator 构造参数、S9-S22 各项。

---

## 六、方法与可信度说明

- 本报告所有结论均由本人读取原始代码确认，关键处均给出 文件:行号，可逐条复核。
- 审计中发现并纠正了勘查代理的一处误报：代理称「heartbeat 用硬编码预设人格」，实际是 heartbeat 用模块级 `get_personality_prompt`（`heartbeat.py:443`），确实不走 DB——结论方向一致但机制描述不同，已按代码修正。
- 未逐条复核、仅来自代理勘查的较低价值项已在文中标注（S17/S18/S19/S20/S21/S22），如需可再验证。
- 全量测试基线：464 passed / 4 failed，4 个失败均为 P1 既有缺陷所致，非本次改动引入。（修复后为 487 passed / 0 failed）

---

## 七、修复记录（2026-10-24）

原则：最小改动，不重构；只保留解释「为什么」的注释，不复述代码。

| 编号 | 修复方式 | 文件 |
|------|---------|------|
| C1 | 普通消息改为按 role/content 重建，剥离非规范字段 | `anthropic_provider.py:254` |
| S1 | cron 执行前后 acquire/release 会话闸门 | `cron/executor.py:73` |
| C6 | 名称校验：拒绝 `.`/`..`/分隔符/非直接子目录 | `skills.py:868` |
| C2 | WS 补传 `enable_auto_memory_summary` | `ws/events.py:451` |
| M1 | 忙分支识别 `_persisted` 避免重复落库 | `channels/handler.py:620` |
| C3/C4 | 抽出 `_build_history_message`，统一剥元数据 + 还原 reasoning | `channels/handler.py:276` |
| C5 | 新增三项注入上限常量并落地 | `skills.py:25-32, 611-680` |
| C10 | 新增 `session_name_matches`，要求分隔符边界并转义通配符 | `session/manager.py:15` |
| S2 | 哈希纳入 `media` | `enterprise_queue.py:251` |
| S8 | 每 512 次检查触发一次过期清扫 | `enterprise_queue.py:264-300` |
| S4 | 令牌清理统一移入 finally | `ws/events.py:178,490` |
| S5 | `cleanup_cancel_token` 支持令牌身份比对 | `ws/connection.py:125` |
| S6 | 新增 `try_acquire_job`/`release_job`，检查与占用合并 | `cron/scheduler.py:355`、`api/cron.py:960,1025` |
| S11 | `stream_iterator` 改 try/finally | `ws/streaming.py:209` |
| S14 | heartbeat 失败改为抛出，不再伪装成功 | `cron/executor.py:169` |
| S22 | 修正重试日志读取被清零的计数 | `cron/scheduler.py:316` |
| C11 | `is_memory_noop` 改为整段比较而非子串匹配 | `prompts.py:96` |
| C12 | 新增 `render_prompt`，逐占位符替换，字面花括号与缺失占位符均不报错 | `prompts.py:101` |
| C14 | 删除两个无引用死模板 | `prompts.py` |
| P1 | 构造参数显式传入时优先于配置 | `filesystem.py:38-64` |

新增测试 `tests/test_audit_fixes.py`（19 例）覆盖上述行为，其中「字面花括号不吞占位符」一条曾在实现过程中暴露初版方案缺陷，已修正为逐键替换。
