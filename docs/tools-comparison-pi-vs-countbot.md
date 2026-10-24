# 工具层对比：pi（4 内置工具）vs CountBot

> 对比对象：`third-party/pi-mono/packages/coding-agent/src/core/tools/`（read/write/edit/bash）
> × `CountBot/backend/modules/tools/`（read_file/write_file/edit_file/exec）
> 方法：两侧源码逐行对读，结论带 文件:行号。承接 `pi-mono-vs-countbot.md`（v2），本报告只聚焦工具层 + 技能形态 + MCP 三个问题。
> 时间：2026-10-24

---

## 0. 先回答三个问题的结论

| 问题 | 结论 |
|------|------|
| 我们四个工具有哪些优势？ | 有，但集中在**契约与安全**层；**上下文经济性**（读/截断策略）明显落后 |
| pi 把技能做成工具了吗？ | **没有**。pi 的技能是 prompt 里的 XML 清单 + 让模型自己用 `read` 读文件，不是工具。我们做成工具更省一次往返，但代价是工具面变宽 |
| MCP 该对齐什么？ | pi 的 **exposure 分级**（4 级）和 **resources 用 3 个通用工具**而非 N 个包装，这两点最值得抄 |

---

## 1. 四个工具逐项对比

### 1.1 读文件：`read` vs `read_file`

| 维度 | pi `read` | CountBot `read_file` |
|------|-----------|----------------------|
| 参数 | `path` / `offset` / `limit`（行号 1-based） | `path` 或 `paths`（批量）/ `start_line` / `end_line` / `show_line_numbers` |
| **默认上限** | **2000 行 或 50KB，先到者生效**（read.ts:12, 96） | **无**（filesystem.py:225） |
| 截断提示 | 给 `[Showing lines 1-2000 of 5321. Use offset=2001 to continue.]`（read.ts:190-192） | 无 |
| 单行超限 | 检测首行即超限，返回可执行的 bash 建议（read.ts:179-183） | 无 |
| 图片 | 支持，且**按模型能力自动缩图**；非视觉模型给降级提示（read.ts:133-152, 79-84） | 无（不支持图片） |
| 批量读 | 无 | **有**（`paths` 数组，filesystem.py:246-260） |
| 行号 | 无（纯文本） | 可选（`1| ` 前缀） |
| 技能联动 | 无 | **禁用技能时拒绝读其 SKILL.md**（filesystem.py:206-212） |
| 取消 | AbortSignal 贯穿读写全程（read.ts:111-120） | 无（见 §1.4） |

**关键差距（这是真问题）**：`end = min(total, int(end_line)) if end_line is not None else total`（filesystem.py:225）。**不传 `end_line` 就把整个文件塞进上下文**。一个 2MB 的日志文件被 read_file 一次读入，直接顶爆窗口。
pi 的 2000 行/50KB 双阈值 + 续读提示是刚需，不是锦上添花。

**我们的优势**：批量读（一次调用读多个文件，省一次往返）、行号显示、技能禁用联动——这几项 pi 都没有。

### 1.2 写文件：`write` vs `write_file`

| 维度 | pi `write` | CountBot `write_file` |
|------|-----------|----------------------|
| **同文件并发保护** | **有**：`withFileMutationQueue` 按 realpath 串行化（write.ts:67-83, file-mutation-queue.ts:32-61） | **无** |
| 父目录 | 自动创建（write.ts:78） | 自动创建 |
| 追加/编辑模式 | 无（只覆盖，文档明确"only for new files or complete rewrites"） | **有**追加、按行编辑等多种模式（filesystem.py:424+） |
| 取消 | 每步 await 后检查，且刻意**不用** abort 监听器抛错以免提前释放队列（write.ts:68-83） | 无 |
| 审计 | 无 | **有** file_audit_logger JSONL + attempt_id 去重 |

**关键差距**：并发写同一文件无保护。当前主循环和子代理可能同时对同一文件写入（子代理有独立 ToolRegistry），后写覆盖先写。pi 的 mutation queue 直接解决这个，实现只有 60 行。

### 1.3 编辑：`edit` vs `edit_file`

| 维度 | pi `edit` | CountBot `edit_file` |
|------|-----------|----------------------|
| 模式 | 只做精确文本替换，一次调用**多块**替换（edits 数组） | 文本替换 **或** 按行范围（两种模式） |
| 重叠检测 | 明确要求不重叠、不嵌套，且对**原文件**匹配而非增量（edit.ts:35-38） | 未见重叠检测 |
| **格式容错** | **强**：edits 传成 JSON 字符串、单对象、旧字段名（oldText/newText）全部兼容（edit.ts:103-134） | 解析错误返回带建议的 VALIDATION 失败（registry 层） |
| **BOM / 换行** | 剥离 BOM 再匹配，还原原换行风格（CRLF/LF）（edit.ts:191-197） | 未见处理 |
| 返回值 | diff + **unified patch** + 首个改动行号（edit.ts:201-210） | 文本提示 |
| 并发保护 | 有（同 write） | 无 |

**关键差距**：pi 的 edit 在**模型输出畸形参数**时的容忍度更高（那三种 shim 是针对 Opus/GLM 实际踩过的坑），且 BOM/换行处理避免了 Windows 编辑过的文件全量 diff。我们的优势是多了按行编辑模式。

### 1.4 命令执行：`bash` vs `exec`

| 维度 | pi `bash` | CountBot `exec` |
|------|-----------|-----------------|
| 超时 | 可选，无默认（bash.ts:42） | 默认 180s（shell.py:95 附近） |
| **输出截断方向** | **保尾部 + 全量落盘 + 给路径**（bash.ts:342-354, 38-62） | **保头部 + 全量落盘 + 给路径**（shell.py:600-640，上一轮已补） |
| 截断提示 | `[Showing lines 4900-5000 of 5000. Full output: /tmp/pi-bash-xxx]` | `完整输出已保存到: temp/exec-output-xxx.log` + read_file 指引 |
| **取消穿透** | **杀整棵进程树**（`killProcessTree`，bash.ts:126-145） | `cancel_token` **只在执行前检查**（registry 层） |
| 进程组 | `detached: true` + 独立进程组，便于整体清理（bash.ts:114） | 未见 |
| 流式进度 | 有 `onUpdate` 节流回调（bash.ts:276-326） | 无（子代理有 event_callback） |
| 安全 | **刻意不设防**，靠可编程 `tool_call` 钩子 | **DANGEROUS_PATTERNS + 自定义 deny/allow + 工作区限制**（shell.py:112-134） |
| 结构化输出 | `output/truncated/full_output_path/exit_code/wall_time_seconds`（bash.ts:56-62） | `ToolResult` 契约（更强，见下） |
| 会话环境变量 | 注入 `PI_SESSION_ID/PI_MODEL` 等（bash.ts:199-209） | 无 |

**关键差距（两个）**：
1. **取消不穿透**：用户点停止，正在跑的长命令继续占用。pi 的 `killProcessTree` 直接杀进程组。
2. **保头部 vs 保尾部**：构建/测试失败的原因在**尾部**。pi 保尾部是对的；我们保头部，配合落盘文件可补救，但要多一次 read_file 往返。

**我们的优势**：安全策略（pi 完全不管，把责任推给扩展钩子）、outcome 契约（exit_code 之外还有 SideEffectState/RetrySafety，支持幂等重放判定）、审计日志。

---

## 2. 我们的结构性优势（pi 完全没有）

这些不在单工具层，但决定了工具层的工程上限：

| 能力 | 位置 | 为什么重要 |
|------|------|-----------|
| **Canonical Tool Outcome** | `execution.py`：ExecutionState / SideEffectState / RetrySafety | 工具失败后能否重试有机器可判的依据。pi 是"回错误给模型让它自己看着办" |
| **OperationLedger 幂等** | registry 层 operation_id 去重 | 网络抖动重放不会把文件写两遍。pi 无此概念 |
| **审计** | file_audit_logger JSONL | 谁在什么时候改了哪个文件，可追溯。pi 遥测默认 noop |
| **工具被禁用时联动** | disabled_tools → `_tool_available` → 提示词区块也消失 | 关了工具，模型连它的描述都看不到，不会产生"我知道有这工具但调不了"的死循环 |
| **批量读** | filesystem.py:246 | 读 5 个文件 1 次往返，pi 要 5 次 |

---

## 3. pi 的结构性优势（我们缺）

| 能力 | 位置 | 差距本质 |
|------|------|---------|
| **双阈值截断（行+字节）** | truncate.ts | 我们只按字符数，且 read_file 没上限 |
| **同文件写入串行队列** | file-mutation-queue.ts | 并发写会丢更新 |
| **取消穿透进程树** | bash.ts:126-145 | 停止按钮对长命令无效 |
| **保尾部截断** | bash.ts:348-353 | 失败原因通常在最后 |
| **edit 三态容错 shim** | edit.ts:103-134 | 模型畸形参数的直接成本 |
| **BOM/换行保持** | edit.ts:191-197 | Windows 文件全量 diff |
| **图片读取 + 按模型缩放** | read.ts:133-152 | 视觉任务能力 |
| **操作可插拔**（ReadOperations 等） | 各工具的 `Operations` 接口 | 同一工具可指向 SSH/远程，无需改工具 |

**最后一条值得单独说**：pi 每个工具都抽出 `operations` 接口（read.ts:48-55、write.ts:27-32、bash.ts:75-94）。这意味着"把文件操作指向远程主机"只需注入一组实现。我们是硬编码本地 `Path` 操作，改造要动工具本体。

---

## 4. 技能：pi 做成了工具吗？

**没有。** 这一点要说清楚，因为容易误解。

pi 的技能机制（`skills.ts:358-388`）：
- 扫描 `SKILL.md`，解析 frontmatter 的 `name`/`description`
- 以 **XML 文本**注入 system prompt：
  ```xml
  <available_skills>
    <skill>
      <name>pdf</name>
      <description>...</description>
      <location>/abs/path/SKILL.md</location>
    </skill>
  </available_skills>
  ```
- 提示词里写清楚："任务匹配描述时，**用 read 工具去读那个文件**"

所以 pi 的技能 = **一份清单 + 一次 read 调用**。技能本身不占工具位，也不进工具注册表。

### 两种形态对比

> **更正（2026-10-24）**：本文初版把 CountBot 描述为"技能即工具"，这是错的。
> 核实 `context.py:172-182` 与 `_skills_usage_hint`（`context.py:398-412`）后确认：
> 我们的技能**同样是文档**——摘要进提示词，模型用 `read_file` 读 SKILL.md。
> 形态与 pi 一致，差异只在注入排版与附加机制。下表已按实际实现修正。

| 维度 | pi（清单 + read） | CountBot（摘要 + read_file） |
|------|------------------|----------------------------|
| 工具数量 | 恒定 4 个 | 恒定 11 个（与技能数无关） |
| 单次调用成本 | 2 次模型往返（read 后执行） | 2 次（同样需先读） |
| 清单排版 | XML `<available_skills>` | Markdown `- name \| title: desc` |
| 上下文占用 | name+desc+**绝对路径**常驻 | name+desc 常驻（有总量与单条上限） |
| 控制粒度 | `disable-model-invocation` 两态 | enabled + requires + always + **manual_only**（三态） |
| 依赖检查 | 无 | **有**（bins/env，`skills.py:286-300`） |
| 自动加载 | 无（必须模型主动 read） | **有**（`always: true` 全文注入，带长度上限） |
| 相对路径解析 | 提示词明确"相对 skill 目录" | 未在提示词中说明 |
| 跨生态 | Agent Skills 标准 XML | 兼容 OpenClaw 目录与标准 frontmatter |

### 谁更好

- **清单排版**：pi 的 XML 带 `<location>` 绝对路径，模型能直接 read，不需要自己拼路径。我们只有 name+desc，模型得先知道技能在哪（提示词另给了一句 `read SKILL.md`，但没给具体路径）。这是 pi 更明确的地方。
- **控制粒度**：我们更丰富。依赖检查、always 自动加载、加上本轮补的 manual_only，覆盖了 pi 只有两态的表达力。
- **上下文经济性**：我们更可控——有总量上限、单条描述截断、always 全文截断；pi 的清单无上限（`skills.ts:377-383` 逐条全量输出）。
- **相对路径解析**：pi 在提示词里显式交代（`skills.ts:372`），我们缺这一句，模型引用技能内相对路径时容易出错。

**结论**：形态本就一致，不需要改造成工具。值得补的是 pi 那两句提示词——**给出绝对路径 + 交代相对路径基准**，成本极低。

---

## 5. MCP：该对齐什么

### 5.1 双方现状

**pi**（`packages/mcp` 是完整客户端 + `core/mcp-servers.ts` 配置层）：

- 客户端能力齐全：OAuth 流程（`oauth/`）、3 种传输（stdio / streamable-http / in-memory）、进度通知（**且进度会重置超时**，client.ts:536-547）、`notifications/cancelled` 双向取消、分页跟随（最多 1000 页，client.ts:357-376）、重复 cursor 检测（client.ts:370-372）
- **工具命名**：`mcp__<server>__<tool>`（mcp-servers.ts:38-41）
- **resources 用 3 个通用工具**：`list_mcp_resources` / `list_mcp_resource_templates` / `read_mcp_resource`（mcp-servers.ts:28-36）——不是每个 resource 一个工具
- **4 级 exposure**：`direct`（直接声明给模型）/ `deferred`（由 `tool_search` 按需加载）/ `codemode`（只在脚本里可调）/ `hidden`（注册但不可达）（mcp-servers.ts:43-52）
- **单工具级覆盖**：`toolExposure: {"delete_*": "hidden"}` 支持通配符（mcp-servers.ts:67-73）
- **服务器描述参与检索**：`description` 用于提示词里的服务器清单、工具检索排序（mcp-servers.ts:62-66）

**CountBot**（`backend/modules/mcp/client.py`，1150 行）：

- 桥接三类：`MCPToolWrapper` / `MCPResourceWrapper` / `MCPPromptWrapper`（client.py:175/261/329）
- **命名**：`mcp_{server}_{tool}`（client.py:179，单下划线）
- **resources 每个一个工具**（client.py:264）——装个有 20 个 resource 的服务器，工具面 +20
- 超时：固定 30s（client.py:176）
- 有 outcome 契约、断线重连、`reconcile_registry_sync`（长连接每轮对齐工具集，ws/events.py:307）

### 5.2 对齐建议（按收益排序）

| 优先级 | 项目 | 现状问题 | 建议 |
|--------|------|---------|------|
| **高** | resources 用通用工具 | 每个 resource 一个工具，工具面随服务器膨胀 | 改成 3 个通用工具（list/templates/read），resource 用 URI 参数 |
| **高** | exposure 分级 | 只有"注册/不注册"二态，MCP 工具全量暴露 | 至少引入 `direct` / `hidden` 两级 + 单工具通配符覆盖。MCP 服务器常有几十个工具，全暴露会淹没内置工具 |
| **中** | 命名分隔符 | `mcp_server_tool` 有歧义（server 名含下划线时无法反解） | 改 `mcp__server__tool`，双下划线可无歧义解析 |
| **中** | 进度通知重置超时 | 长任务 MCP 工具 30s 必死 | 收进度通知则重置计时器（pi 做法） |
| **中** | 服务器描述进提示词 | 有 `description` 但不参与呈现/检索 | 提示词里列出服务器清单+描述，模型才知道去用 |
| **低** | 双向取消 | 有取消检查但不下发 `notifications/cancelled` | 补协议级取消，让服务端及时释放 |
| **低** | OAuth | 未支持 | 多数远端 MCP 需要，但可延后 |

**不建议对齐的**：`codemode` 曝光（需要脚本执行环境，我们是服务端多会话，语义不符）、`deferred`（需要 `tool_search` 工具配合，属于另一套体系）。

---

## 6. 行动清单

### P0（正确性/资源风险，建议马上做）

1. **read_file 加双阈值**：默认 2000 行 / 50KB，超限给 `offset` 续读提示。当前无上限直接读全文件，是上下文爆炸的单点风险。
2. **exec 取消穿透**：`cancel_token` 注册回调 → 杀进程组。停止按钮现在对长命令无效。
3. **write/edit 同文件串行队列**：照 pi 的 `withFileMutationQueue`（60 行），防并发丢更新。

### P1（体验/成本）

4. **exec 截断改保尾部**：失败原因在尾部，现在保头部要靠落盘文件补救，多一次往返。
5. **edit 加 BOM/换行保持**：避免 Windows 文件全量 diff。
6. **MCP resources 改 3 个通用工具** + 命名改双下划线。

### P2（架构）

7. **MCP exposure 分级**（direct/hidden + 通配符覆盖）。
8. **工具 operations 可插拔**：把文件操作抽接口，为远程/SSH 场景留口。
9. **技能加"仅手动调用"档**：借鉴 pi 的 `disable-model-invocation`，补 enabled/disabled 之外的第三态。

---

## 7. 一句话总结

工具层的差距**不在功能多少，而在边界处理**：pi 的四个工具每个都在处理"模型会犯错、文件会很大、用户会中途停止"这些真实边界（双阈值截断、写入串行、取消穿透、参数容错 shim、BOM 保持）；我们的工具功能更全（批量读、多模式编辑、安全策略、outcome 契约、审计），但在**资源上限和中断语义**上有实质缺口，其中 read_file 无上限是能在线上直接造成故障的一条。

技能不是工具这件事，pi 的选择更克制——用"清单 + read"把工具面恒定在 4 个；我们做成工具换来少一次往返，代价是工具面随技能数增长。三种形态各有其位，最该补的是中间档"仅手动调用"。

MCP 上 pi 明显更成熟（OAuth、进度、取消、exposure 分级），我们该抄的是 **resources 用通用工具** 和 **exposure 分级**这两条与工具面直接相关的设计。

---

## 8. 落实情况（2026-10-24 实施）

> 实施原则：借思路不照抄，按本项目既有结构做最小落点；只保留解释「为什么」的注释。

### 已实施

| 项 | 做法 | 与本项目的结合点 |
|----|------|-----------------|
| **read_file 双阈值** | 默认 2000 行 / 50KB，先到者生效；超出给 `start_line=next` 续读提示；批量读另设总量预算 `2×max_chars` | 保留我们独有的**批量读**与行号显示；上限做成构造参数可覆盖；`max_chars` 只按字符预算裁行，避免单行超长文本绕过 |
| **写并发串行** | 新增 `path_lock.py`：进程级 `PathLockRegistry` + `path_write_lock` 异步上下文 | 未照抄 pi 的 Promise 链，改用 asyncio.Lock + 引用计数回收；write 与 edit 各自在**读-改-写整段**加锁，而非只在写入瞬间 |
| **exec 取消穿透** | `set_cancel_token` + 取消回调 → `_terminate_process` 按进程组 SIGTERM/SIGKILL；`start_new_session=True` | 沿用 registry 已有的 `set_cancel_token` 约定；用一次性哨兵避免长会话回调累积误杀后续命令 |
| **exec 保尾部截断** | 头部保留 25%、尾部保留 75%，中间标注省略字符数 | 不直接抄"纯保尾"：开头的提示信息也有价值，故做比例切分；仍保留落盘 artifact |
| **MCP resources 通用化** | `MCPResourceCatalogWrapper`：每个服务器 1 个 `mcp__server__read_resource`，`uri` 参数选择 | 描述里列出资源清单（超 20 条截断），保留我们的 outcome 契约 |
| **MCP 命名** | `mcp__server__tool` 双下划线 | server/tool 名内的单下划线不再产生歧义 |
| **技能第三态** | `disable-model-invocation` / `disable_model_invocation` → `manual_only`：不进摘要清单、不被 always 自动注入，但仍在末尾点名以免用户以为技能丢失 | 兼容 pi 与开放标准两种写法；保留我们独有的依赖检查与 always 机制 |

### 有意未做

| 项 | 原因 |
|----|------|
| MCP exposure 四级分级 | `codemode`/`deferred` 依赖脚本池与 `tool_search` 体系，本项目无对应基础设施；直接引入会带来无法兑现的配置项。当前以 `enabled_tools`/`exclude_tools` 做粗粒度控制已够用 |
| MCP OAuth | 本项目 MCP 以本地 stdio 为主，远端鉴权需求未出现，待有实际场景再做 |
| edit 的 BOM/换行保持 | 与现有 `read_text`/`write_text` 的 UTF-8 约定一致即可；引入 BOM 往返会增加路径分支，收益在 Linux 部署下有限 |
| 工具 operations 可插拔 | 属架构级改造（远程/SSH 场景），非边界加固范畴 |

### 验证

- 新增 `tests/test_tool_boundaries.py`（12 例：读上限、字符预算、批量预算、锁串行、并发追加、并发编辑、取消穿透、子进程终止）
- 新增 `tests/test_skill_manual_only.py`（6 例）
- 全量：**505 passed / 0 failed**
- 端到端：10000 行文件读取返回 2000 行 + `start_line=2001` 提示；`sleep 30` 在取消后 8 秒内终止
