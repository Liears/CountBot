# pi-mono vs CountBot 深度对比（v2，代码级）

> 对比对象：`third-party/pi-mono`（badlogic/earendil-works，TS）× `CountBot`（Python FastAPI + Vue）
> 方法：两侧源码逐模块勘查，所有结论带 文件:行号 证据。v1 报告的表层判断已全部复核，两处修正见 §0。
> 时间：2026-10-23

---

## 0. 对 v1 的两处修正

1. **当前快照中没有 web-ui 包**。实际包列表：`agent, ai, chord, coding-agent, codemode, durable, env, evals, mcp, telemetry, tui`。v1 引用的"pi-web-ui Web Components"来自旧资料，pi 现在的 UI 策略是 TUI 组件 + 会话导出静态 HTML（`coding-agent/src/core/export-html/`）+ headless RPC 模式三件套。
2. **"跨供应商 thinking 转文本"是误传**。pi 的实际做法是**直接丢弃**跨模型的 thinking 块（签名只对产出的模型有效，伪造文本反而危险）：`packages/ai/src/api/transform-messages.ts:128-137`。

---

## 1. 哲学定位

| | pi | CountBot |
|---|---|---|
| 一句话 | 少注入、全透明、核心拒绝功能、一切靠扩展钩子 | 多能力、可裁剪、强契约（outcome/审计）、服务常驻多渠道 |
| 上下文观 | "模型看到什么必须可见可查"，prompt 是事件流 | "注入要可控"，prompt 是拼出来的字符串 |
| 历史观 | append-only 事件树，永不删数据，投影出上下文 | 线性表 + 游标，压缩即生成摘要缓存 |
| 规模 | 4 工具、prompt <1000 tok | 13+ 工具、prompt ~1,619 tok + 工具 ~4,600 tok |

两边不冲突，但各自有对方没有的"杀手细节"。下面逐模块对照。

---

## 2. Agent 循环：状态机、事件、steering

### pi（packages/agent/src/agent.ts + agent-loop.ts）

- **壳与算法分离**：有状态 `Agent`（单飞互斥 `activeRun`，agent.ts:182-186）+ 无状态纯循环 `agent-loop.ts`。
- **11 类事件三层生命周期**（types.ts:516-539）：`agent_start/end` → `turn_start/end` → `message_start/update/end` → `tool_execution_start/update/end`，订阅者按序 await，`agent_end` 全部 settle 才算 idle。
- **双队列 steering**（这是 CountBot 最大的机制空白）：
  - `steer()`：执行中插话，注入点在**每个 assistant turn 的全部工具执行完之后**轮询（agent-loop.ts:303-307，注释明确"不会打断正在执行的工具"），下一轮 provider 请求前经 `declareToolChanges()` 包成 system 消息入上下文；
  - `followUp()`：只在 agent 自然停下的外层循环点 drain（agent-loop.ts:310-317）；
  - 队列模式 `all | one-at-a-time`（types.ts:55）；`prepareNextTurn`（如压缩）执行完会**再轮询一次** steering，防长操作期间漏话（agent-loop.ts:210-215）。
- **钩子保守策略**：`beforeToolCall` 只能 block/terminate **不能改参**（types.ts:66-74）；`afterToolCall` 可整体替换返回值，逐字段浅覆盖不深合并（agent-loop.ts:880-911）；改参数走 coding-agent 层 `tool_call` 事件（原地 mutate `event.input`）。批次终止条件：**整批结果全部 terminate 才停**（agent-loop.ts:700-702）。
- abort 会合成 `stopReason:"aborted"` 消息并补齐事件序列（agent.ts:542-558）。

### CountBot（backend/modules/agent/loop.py）

- `process_message` async 生成器，`max_iterations=25` 且与 `total_tool_calls` 共用预算（loop.py:377-390）。
- **三层重试模型是亮点**：provider 级 key 轮换（`_try_key_rotation`，仅认证/限流，MAX 3 次，loop.py:809）；工具级重试带 `RetrySafety + SideEffectState` 状态机门禁（必须 SAFE + 非 UNKNOWN 才允许，loop.py:537-544）；工具 60s 超时，`spawn/workflow_run/external_coding_agent` 三个自管闹钟工具豁免（loop.py:43）。
- **steering：完全没有，且有两处竞态**：
  - WS 同连接串行（ws/events.py:654-715），但**没有按 session 的互斥**，第二个连接可并发触发同 session 的第二轮 `process_message`；
  - IM 渠道 `asyncio.create_task(handle_message)` 直接并发（channels/handler.py:470-472），`_active_tasks[session_id]` 直接**覆盖**旧任务引用（handler.py:612）——工具执行期间来新消息会并发跑，历史交错写入。目前唯一的"排队"在前端 `useQueuedMessages.ts:87-107`（isStreaming 时 pendingMessages）。

**对照结论**：CountBot 的重试/安全契约比 pi 细（pi 框架层连工具重试都没有，参数校验失败只回错误结果让模型自己重发）；但 pi 的 steering 双队列是"人在环上"产品的刚需——**IM 渠道用户连发两条消息时 CountBot 的行为是未定义的**，这不是功能差距而是正确性 bug 级空白。

---

## 3. 供应商层：协议收敛 vs 适配器 + 自学习

### pi（packages/ai/）

- **40+ provider 目录 × 仅 10 种 API 协议实现**（types.ts:64-74, all.ts:4-46）：openai-completions/responses、anthropic-messages、bedrock-converse、google-generative-ai/vertex、mistral、codex、pi-messages。多供应商复用同一协议代码。
- **跨供应商修补单点化**：`transform-messages.ts` 集中处理——thinking 跨模型丢弃、非视觉模型图片降级占位（:62-84）、toolCall id 归一化（OpenAI Responses 450+ 字符 id → Anthropic 64 上限，:87-94）、孤儿 toolCall 合成 `"No result provided"`、error/aborted 消息不重放。
- **工具参数不自动重试**：TypeBox 校验 + 三层宽容矫正（删可选字段 null、Value.Convert、手写 coercion，validation.ts:59-131），报错带路径回显给模型让它重发；HTTP 层才有重试（provider-retry.ts，镜像官方 SDK 策略）。
- `stopReason:"length"` 的截断消息里所有 toolCall 一律判失败不执行（agent-loop.ts:276-279）——防止执行半截 JSON 参数。

### CountBot（backend/modules/providers/）

- `LLMProvider` 基类 + `StreamChunk` 统一载体（base.py:17-50），OpenAI/Anthropic 两条手写流。
- **reasoning 全链路保真是亮点**：多形状提取（reasoning_content/reasoning/thinking + model_extra，openai_provider.py:427-442）→ 存 message_context（events.py:411-413）→ 下轮注入（context_service.py:36-41）；Anthropic 侧连 thinking+signature 原生 content blocks 都完整往返（provider_payload roundtrip，anthropic_provider.py:628-635 → 213-219），切 OpenAI 时被白名单过滤（openai_provider.py:495-521）。**这一点比 pi 强**：pi 换模型就丢 thinking，CountBot 同模型往返不丢。
- **兼容性自学习缓存是独有设计**：按 `provider|api_base|model` 记忆"该上游要 strip reasoning / 简化 tool schema / 固定 temperature"等 hint 并自动套用（openai_provider.py:895-1035）。pi 没有对应物——它靠协议收敛回避了这个问题，但 CountBot 面对的国产/中转 OpenAI 兼容上游长尾恰恰需要这种脏活。
- 流式都有"未产出内容可整流重试、已产出优雅截断 length"（openai_provider.py:332-353）。
- 空白：`StreamChunk.usage` **没有任何消费者**（§8）；OpenAI 侧没有 Anthropic 式 cache_control 断点。

**对照结论**：pi 用"协议收敛"把长尾问题归一，CountBot 用"自学习 + 原块往返"逐个驯服长尾——后者更脏但更贴合国产上游现实，是应保留的差异化优势。值得抄的是 pi 的两条纪律：`stopReason=length` 时废掉全部 toolCall；孤儿 toolCall 合成显式错误结果。

---

## 4. 工具系统

| 维度 | pi | CountBot |
|---|---|---|
| 输出截断 | read：头部 2000 行/50KB + `Use offset=N` 续读提示（read.ts:184-193）；bash：**保尾部** 2000 行/50KB，**全量输出落盘临时文件**并把路径给模型（bash.ts:342-354） | 分散各工具：exec 10000 字符（shell.py:501-507）、web_fetch max_chars+truncated 标志、子代理结果只存 500 字符；**Registry 层无统一策略** |
| 写并发 | `withFileMutationQueue` 按路径串行化同文件写（write.ts:67-83） | 无 |
| 超时/取消 | 框架无统一超时，AbortSignal 贯穿 + bash `killProcessTree` 杀整棵进程树（bash.ts:126-145） | 工具 60s wait_for；`cancel_token` **只在执行前检查**（registry.py:324-339），执行中不可中断 |
| 安全 | 刻意不设防（bash 无命令过滤、read 无目录白名单），安全上移到可编程 `tool_call` 钩子（示例 `confirm-destructive.ts`） | exec 有 DANGEROUS_PATTERNS + 自定义 deny/allow + file_audit_logger JSONL 审计（按 attempt_id 全局去重） |
| 工具可见性 | 5 级 exposure：`direct/model-only/codemode/deferred/hidden`（extensions/types.ts:516） | 二态：注册/被 disabled_tools 卸载 |
| 嵌套调用 | `ctx.executeTool` 子调用走同样校验/钩子但只记 nestedCalls 不进 transcript（agent-loop.ts:821-829） | 子代理独立 ToolRegistry，任务落 tasks 表 |
| 畸形参数 | edit 工具自带三种格式兼容 shim（edits 传成 JSON 字符串/单对象/旧字段名，edit.ts:103-134） | 解析错误塞特殊键 → Registry 返回带"分块写文件建议"的 VALIDATION 失败（registry.py:241-290） |

**对照结论**：CountBot 的审计/幂等（OperationLedger：operation_id 重放去重，registry.py:412-422）比 pi 系统化；pi 有三个马上可抄的小件——**bash 全量输出落盘给路径**（CountBot exec 超长输出直接截掉，模型想看后半段没办法）、**write 按路径串行队列**（CountBot 并发写同文件无保护）、**取消穿透到执行中**（CountBot 只在执行前查 cancel_token）。

---

## 5. 上下文管理：这是差距最大的模块

### pi 的三板斧

1. **System prompt = transcript 内的事件流**。prompt 拆成具名节（XML 标签），`SystemMessage.sections` 直接写进会话；后续变化**追加一条带 sections patch 的新 system 消息**（变了→新值，没了→null），patch 为空就不发任何消息（agent-session.ts:1759-1770 + system-prompt.ts:217-229）。transcript 永不改写，重放即恢复。
2. **Anthropic cache_control 显式断点**：`cache_control:{type:"ephemeral"}` 打在最后一个 system/user 消息和最后一个 tool 上（anthropic-messages.ts:1547-1568）；`cacheRetention: short|long|none` 选项；OpenAI 侧靠前缀稳定。
3. **cache warmer（罕见实现）**：`core/cache-warmer.ts` 按 TTL 的 90% 定时重放请求（max_tokens=1）续缓存，**预算制**——只有预期省钱 ≥ $0.05 才 warm，还要处理 Anthropic budget-thinking 重放破坏缓存 key 的边界（isReplayable），warm 消耗以 `usage entry kind:"cache_warm"` 归因。

### CountBot 现状

- 静态前缀策略（system 纯静态 + 动态内容进首条 user，context.py:478-525）方向正确，但**开关一变 = 前缀整体变 = cache 全失效重建**，无任何缓解手段；
- 无 cache_control 断点（Anthropic 流走 raw httpx 也一样没打）；无 cache warmer；**无 usage 数据所以连"缓存命中率值多少钱"都算不出来**。

### compaction 对比

| | pi | CountBot |
|---|---|---|
| 触发 | `contextTokens > contextWindow - 16384`（留回复余量），三来源 manual/threshold/overflow | 条数驱动：IM >15 条、短期摘要 >max_history_messages、MEMORY.md 沉淀 ≥30 条/15000 字符——**无 token 级窗口概念** |
| 切点 | 必须落在 user/assistant 消息（`isCutPointMessage` 排除 toolResult，防切断工具对，compaction.ts:351-364）；turn 中间还会拆 turnPrefixMessages 单独摘要 | IM 压缩 `keep_recent=10` 条，**可能把 tool_call/tool_result 对切断**（handler.py:2248-2274） |
| 数据安全 | **投影式替换**：被摘要的旧 entry 物理保留在 JSONL，`buildContextEntries()` 重放时窗口化；compaction entry 冻结当时的完整 SystemMessage | 摘要缓存进 sessions 表四字段 + 游标；原始消息还在，但"哪个视角构建上下文"散在三套机制里 |
| 结构化模板 | 目标（增量）/决策（只增+理由）/路径/函数名/错误信息原文分节强制保留 | ConversationSummarizer 只有 char_limit=2000，无结构约束 |
| 部署形态 | 一套机制，CLI/Web/渠道通用 | **三套并行且不统一**：web 端短期摘要（落库缓存）、IM 端请求时同步压缩、MEMORY.md 后台沉淀——同一条会话走不同链路行为不同 |

**对照结论**：这是 CountBot 最该整体重做的模块。方向不是抄代码而是抄三个原则：**token 级触发**（不是条数）、**切点保护工具对**（IM 压缩现在可能制造孤儿 tool_result，是潜在线上 bug）、**投影式替换**（历史永不物理消失，多链路共享一个投影函数而不是三套机制）。

---

## 6. 会话存储：线性表 vs 事件树

pi 的 JSONL 是 **11 种 entry 类型的 append-only 树**（session-manager.ts:183-194）：message、thinking_level_change、model_change、usage、compaction、branch_summary、custom（不进上下文的扩展状态）、custom_message、context_edit（对早前 entry 的追加式改写，不破坏 append-only，:519-540）、label、session_info。每条带 id/parentId/timestamp。

- `branch()` 只是把 leaf 指针移过去，**不复制不删除**（:1579-1584）；
- `createBranchedSession()` fork 成新文件带 parentSession 指针，处理 compaction 的 firstKeptEntryId 重映射（:1632-1727）；
- fork 被放弃的路径自动生成 branch_summary（"交接摘要"），与 compaction 是两个正交概念。

CountBot 的 messages 表是线性 + 游标。**借鉴成本最低收益最大的一步**：加 `parent_id` 列（默认 NULL=尾随），前端做"从这条重新来"。pi 的 `context_edit`（append-only 改写历史）思想还可以解决 CountBot 一个潜在需求：编辑/撤回消息而不破坏重放语义。

---

## 7. 扩展能力：约 40 个钩子 vs 0 个钩子

pi 的 `ExtensionAPI`（extensions/types.ts:1596-1922）：

- 事件钩子约 40 种：生命周期（session/compact/shutdown）、上下文（`context` 可改消息、`before_agent_start` 可换整个 systemPrompt）、供应商（`before_provider_request` 可换 payload、`before_provider_headers`、`after_provider_response`）、工具（tool_call/tool_result）、输入（input/user_bash/model_select）；
- 注册面：工具/命令/快捷键/消息渲染器/主题/Provider（含 OAuth）/虚拟模型/MCP 服务器；
- jiti 零构建直载 .ts，`moduleCache:false` 支持热 reload（loader.ts:562-593）；examples/extensions/ 下 40+ 个示例（confirm-destructive、git-checkpoint、custom-compaction、handoff…）。

CountBot：**没有任何第三方扩展点**。新工具只能改 `register_all_tools`，新编排只能改引擎本体；团队触发靠硬编码的 `@team-name` 文本检测（context.py:486-494）。技能系统兼容 Claude Code/OpenClaw frontmatter（含 `~/.openclaw/skills` 目录，skills.py:493-510）算是唯一的生态开放面。

**建议**：不必照搬 40 个钩子，但 CountBot 有三个天然的钩子位现在全是 if-else 硬编码——**工具执行前（确认流）、provider 请求前（改 payload）、消息入口（渠道路由规则）**。先开这三个，`@团队` 检测和未来的敏感词过滤都能下沉成"内置扩展"。

---

## 8. 可观测性：usage 一等公民 vs 采集了没人要

- pi：usage 内嵌每条 assistant 消息；非对话消耗（摘要/cache warm）以 `usage entry + kind 归因` 追加进会话树；`combineUsage`/`getUsageCostBreakdown` 按 provider/model 分桶（usage-totals.ts:63-99）；TUI footer 实时显示 context 占用百分比；cache warmer 连"warm 这一次值不值 $0.05"都算。
- CountBot：`StreamChunk.usage` 字段存在，Anthropic 侧连 cache_read/cache_write 都采到了（anthropic_provider.py:1035-1054），**但主循环从不读它，无表、无展示、无成本**（全库 grep 确认零消费）。

**这是最干净的空白**：链路每一环都已存在（采集✓ → 只差主循环读一个字段 → 落一张 usage 表 → 设置页一条曲线）。而且没有它，注入开关"到底省了多少 token"、cache warmer"值不值"都永远停留在估算——你做开关体系的初衷就是省 token，现在没有数字闭环。

---

## 9. CountBot 反超 pi 的点（不要在"学 pi"时丢掉）

1. **Canonical Tool Outcome 契约**：ExecutionState/SideEffectState/RetrySafety 贯穿主循环、子代理、MCP、审计四端，OperationLedger 幂等重放去重——pi 完全没有这层，它的错误处理是"回给模型让它自己看着办"。
2. **兼容性自学习缓存**（openai_provider.py:895-1035）：对国产/中转上游的脏活，pi 的协议收敛路线覆盖不到。
3. **reasoning 全链路保真**：pi 跨模型直接丢 thinking，CountBot 同模型往返不丢。
4. **服务化能力**：9 个 IM 渠道热重载（`_reload_injection_runtime` 单点）、cron/心跳、多端共享组件——pi 是纯 CLI，这块它反过来没得比。
5. **团队编排**：pipeline/graph(DAG+条件跳过)/council(交叉评审) + WS 实时推送 + ChildOutcomeError 结构化失败聚合（workflow.py），比 pi "用 bash spawn 自己"的野路子正式得多。
6. **审计**：file_audit_logger JSONL + attempt_id 去重；pi 的遥测默认 noop。

---

## 10. 行动清单

### P0（正确性，本周该做）
1. **session 级互斥 + 消息排队**：修 IM `_active_tasks` 覆盖竞态与 WS 跨连接并发；顺路实现最小 steering（工具批次间检查队列，照 pi 的注入点：turn 末尾、工具全部完成后）。今天不修，多消息用户迟早撞上历史交错。
2. **IM 压缩切点保护**：`keep_recent` 切分时校验不切断 tool_call/tool_result 对（照 pi `isCutPointMessage`）。

### P1（闭环，下周）
3. **usage 持久化**：主循环消费 `chunk.usage` → 新表 usage(session_id, model, prompt/completion/cache_read/cache_write/cost, kind, created_at) → 设置页曲线。注入开关的收益从此有实数。
4. **请求预览接口**：`build_messages` 产物加 debug 端点 + 前端"查看本轮请求"面板（system prompt 全文/工具列表/首条 user 消息）。终结"注入到底进没进"类问题。
5. **exec 输出落盘**：超长输出存 temp/ 文件并把路径回给模型（照 pi bash），配合 read_file 模型可自查。

### P2（架构，本月/下月）
6. **System prompt 分节化**：InjectionConfig 一开关一节，transcript 记 sections，变更走 patch；顺带获得"按节展示"能力。Anthropic 链路补 cache_control 断点。
7. **统一 compaction**：合并三套机制为一个"上下文投影"函数，token 级触发（reserve 16k），结构化摘要模板。
8. **会话树**：messages 加 parent_id + 前端"从这里重来"。
9. **三个扩展钩子位**：工具执行前 / provider 请求前 / 消息入口，先服务内部（确认流、@团队下沉），再考虑开放。

---

## 11. 一句话总结

第一轮的结论"学它的透明与分节"只对了三分之一：**代码级看完，pi 真正值钱的是三个工程纪律——历史 append-only + 投影（永不删数据）、上下文变化表达为 patch 而非重写、token 经济学算到分币（cache warmer + usage 归因）**；而 CountBot 真正值钱的（outcome 契约、自学习兼容、多渠道服务化）pi 一样没有。两边互相抄，CountBot 优先补的不是"更像 pi"，而是 steering 竞态、usage 闭环、切点保护这三个 P0/P1。
