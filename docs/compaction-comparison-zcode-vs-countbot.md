# 压缩机制对标：ZCode vs CountBot

> 对比对象：`ZCode-main/packages/`（TypeScript）× `CountBot/backend/`（Python）
> 方法：读 ZCode 的协议类型与运行时常量，反推其设计意图；逐项对照 CountBot 现状
> 时间：2026-10-24

---

## 0. 核心结论

**ZCode 的压缩是「按 token 水位、可观测、可干预」的工程系统；CountBot 是「按条数触发、结果不可见、只有降级」的尽力而为。**

差异不在算法复杂度，在**三个设计前提**：

| 维度 | ZCode | CountBot |
|------|-------|----------|
| 触发依据 | **token 水位**（`usedTokens / maxTokens`，带 `autoCompactThresholdTokens`） | **消息条数**（>15 条 / >30 条） |
| 过程可观测 | 完整状态机 + 前后 token 计数 + 阶段标记 | 只有一条 `logger.info` |
| 失败处理 | 有 `retrying` / `skipped` / `interrupted` 等状态，可重试 | 静默返回原历史 |

---

## 1. ZCode 的压缩设计（从类型定义反推）

### 1.1 触发是 token 驱动的

```typescript
contextWindow: {
  usedTokens: number,
  maxTokens: number,
  autoCompactThresholdTokens: number | null,   // ← 关键
}
```

压缩阈值以 **token** 表达，而不是消息条数。这解决了「短消息多」与「长消息少」两种极端：20 条长文档消息可能已经爆窗口，200 条短对话却还很宽松。

**CountBot 现状**：`should_compress()` 只看 `len(messages) > 15`。一条粘贴的大日志就足以撑爆窗口，而 100 条「好的」「继续」反而触发压缩——**判据与真正的风险源（token 体积）无关**。

### 1.2 触发来源（trigger）有五种

```typescript
type ZCodeTimelineTrigger = "manual" | "auto" | "reactive" | "partial" | "session_memory";
```

- `manual` —— 用户显式 `/compact`
- `auto` —— 达到阈值自动
- `reactive` —— 上游报错（超长）后被动触发
- `partial` —— 只压一部分
- `session_memory` —— 沉淀到长期记忆

**CountBot 现状**：只有「自动」一种，且分散在三处（IM 渠道、短期摘要、MEMORY 沉淀），没有统一的触发语义，也没有用户手动触发入口。

### 1.3 压缩有明确的阶段（phase）

```typescript
type ZCodeContextCompactionTimelinePhase =
  | "standalone_turn"   // 独立一轮，只做压缩
  | "pre_request"       // 发请求前压
  | "mid_turn"          // 一轮中间压（工具批次之间）
  | "reactive";         // 出错后压
```

`mid_turn` 尤其值得注意：**工具调用轮次之间也能压缩**，因为一轮可能跑几十个工具、历史暴涨。

**CountBot 现状**：只在「请求开始前」压缩一次。长任务跑到一半历史膨胀，无处理。

### 1.4 压缩结果带前后水位

```typescript
preCompactTokenCount?: number,
postCompactTokenCount?: number,
truePostCompactTokenCount?: number,   // ← 注意这个 "true"
```

`truePostCompactTokenCount` 的存在说明：**压缩后估算的 token 与实际差很多**，所以单独记录真值。这是被现实教育过的设计。

**CountBot 现状**：完全不记录，无法判断压缩是否真的有效。

### 1.5 失败是可重试的状态，不是静默降级

```typescript
type ZCodeTimelineStatus =
  | "started" | "retrying" | "skipped"
  | "completed" | "failed" | "interrupted";
```

配合 `attempt` / `maxAttempts` 字段，压缩是个**有重试预算的操作**。

**CountBot 现状**：`compress()` 失败返回 `None`，调用方「沿用原历史」——**用户和运维都不知道压缩失败了**。历史继续膨胀，直到请求真的被上游拒绝。

### 1.6 `inputId` 合并乐观 UI 与真实事件

```typescript
/** `/compact` 本地会先渲染 optimistic 横条，agent lifecycle 事件稍后才到。
 *  用 inputId 把两者合并，避免同一次压缩先显示"正在压缩"再额外追加一条"已压缩"。 */
inputId?: InputId,
```

这条注释说明压缩是**用户可见的过程**，需要 UI 状态去重。

### 1.7 上下文占用按来源分桶

```typescript
type ZCodeContextUsageBreakdownSource =
  | "system_prompt" | "meta_user_context" | "skills"
  | "tool_prompt" | "system_tool_schemas" | "mcp_tool_schemas"
  | "messages";
```

**这是最值得借鉴的一点**：不只告诉你「用了 80%」，还告诉你「其中 system prompt 占 12%、skills 占 25%、messages 占 50%」。压哪里一目了然。

**CountBot 现状**：`request_preview` 能给出 system prompt 各区块字符数，但**不与 token 水位关联**，也没有 messages/tools 的分桶。

---

## 2. CountBot 现状（三套并行机制）

| 机制 | 位置 | 触发 | 问题 |
|------|------|------|------|
| IM 渠道压缩 | `handler.py` `_get_session_history` | 消息 >15 条 | 条数驱动；失败静默 |
| Web 短期摘要 | `context_service.py` | >30 条 或 >15000 字符 | 与上面重复；语义不同 |
| MEMORY 沉淀 | `context_service.py` | ≥3 条待总结 | 沉淀到长期记忆，不释放上下文 |

**三套机制互不知情**：同一条会话走 IM 和走 Web，压缩行为完全不同。这是历史上「三套并行」遗留的结构问题。

---

## 3. 差距排序（按投入产出比）

### P0：token 级触发（替代条数触发）

**为什么最该做**：条数判据与真实风险（token 体积）**根本无关**。这是唯一会导致「压缩了但还是爆窗」的缺陷。

**怎么做（最小改动）**：
```python
def estimate_tokens(messages) -> int:
    """粗略估算：中文 1 字 ≈ 1 token，英文 4 字符 ≈ 1 token。"""
    total = 0
    for m in messages:
        content = str(m.get("content") or "")
        cjk = sum(1 for ch in content if "\u4e00" <= ch <= "\u9fff")
        total += cjk + (len(content) - cjk) // 4
        total += 4  # role 等结构开销
    return total

def should_compress(messages, threshold_tokens=80_000) -> bool:
    return estimate_tokens(messages) > threshold_tokens
```

保留条数作为**兜底**（避免极短消息堆积），但主判据换成 token。

### P1：压缩结果可观测

**为什么**：现在压缩失败无感知，是最危险的「看起来在工作」状态。

**怎么做**：
- `compress()` 返回结构化结果，而不是 `None`/列表二选一：
  ```python
  @dataclass
  class CompactionResult:
      ok: bool
      reason: str            # "no_safe_cut" / "provider_error" / "summary_too_long" / "ok"
      before_tokens: int
      after_tokens: int
      summarized_count: int
      kept_count: int
  ```
- 记录到 `model_usage` 或独立表，`/api/system/usage` 能查到「压缩了几次、省了多少 token、失败几次」

### P2：`mid_turn` 压缩（长任务中途）

**为什么**：CountBot 的 Agent 循环单轮可跑 25 次迭代、几十个工具调用，历史在**一轮之内**就可能爆掉。当前只在请求入口压一次。

**怎么做**：在 `loop.py` 的工具批次间隙检查 token 水位，超过阈值触发一次压缩（复用 `HistoryCompactor`）。这个位置已有插话注入逻辑，加一个检查点成本很低。

### P3：上下文占用分桶

**为什么**：`request_preview` 已有各区块字符数，但用户真正想知道的是「**这些占了多少 token、压哪块最划算**」。

**怎么做**：在 preview 返回里增加：
```python
"usage_breakdown": [
    {"source": "system_prompt", "chars": 4285, "est_tokens": 2100},
    {"source": "tools", "chars": 8206, "est_tokens": 3200},
    {"source": "messages", "chars": 45000, "est_tokens": 28000},
]
```
并对照当前模型的 `max_tokens` 给出百分比。

### P4：手动触发入口

给用户一个 `/compact` 或 API，让「我觉得太长了」这种直觉能立刻生效，不用等阈值。

---

## 4. 明确不该照搬的

| ZCode 做法 | 为什么不抄 |
|-----------|-----------|
| 五种 trigger + 七种 status 的完整状态机 | 那是为多端 UI（desktop/web/mobile）实时同步设计的。CountBot 单端为主，先把 P0/P1 做完再谈 |
| `inputId` 乐观 UI 合并 | 同上，属于前端交互层，与压缩算法无关 |
| `session_fork` / `branch` 相关 | 会话树是完全独立的特性，不是压缩范畴 |

**判断标准**：抄「让压缩更准确、更可观测」的部分；不抄「为多端协同 UI 服务」的复杂度。

---

## 5. 建议的实施顺序

```
第 1 步：token 估算 + 阈值触发（改 should_compress）
         ↓ 风险最低，收益最大，约 30 行
第 2 步：CompactionResult 结构化返回（改 compress 签名）
         ↓ 让失败可见，为后续观测打基础
第 3 步：压缩记录落库 + usage 接口展示
         ↓ 有了数据才能判断阈值设得对不对
第 4 步：mid_turn 压缩（改 loop.py 工具批次间隙）
         ↓ 依赖 1、2 完成
第 5 步：preview 增加 token 分桶
         ↓ 纯展示，可最后做
```

**每步都必须先补测试**（这是前面几轮反复验证过的教训）。

---

## 6. 一句话总结

ZCode 把压缩当成**有水位、有状态、有账本**的工程能力；CountBot 把它当成「历史太长就砍一刀」的临时代码。

最该补的不是算法，是**判据（token 而非条数）和可见性（失败要能被发现）**——这两件事加起来不到 100 行，却能消除当前最大的风险：压缩以为自己在工作，实际早就失效了。

---

## 7. 已实施（2026-10-24）

按上文 P0/P1/P3 落地，测试 589 → **650 passed**。

### 7.1 P0：token 级触发

`history_compactor.py` 新增：

```python
def estimate_tokens(text: str) -> int:
    """CJK 1:1、其他 4:1 估算，误差 ±20%，够用于量级判断。"""

def estimate_messages_tokens(messages) -> int:
    """含 role/分隔符/tool_calls 结构开销。"""

def should_compress(messages, threshold=15, *, token_threshold=None):
    """token 为主判据，条数仅兜底。"""
```

`HistoryCompactor` 新增 `token_threshold=80_000` 参数。传 `None` 时保持旧的纯条数行为，兼容既有调用方。

**验证**：一条 100 万字符的超长消息（2 条消息）在 `token_threshold=1000` 下正确触发压缩——这是旧条数判据完全做不到的。

### 7.2 P1：结构化结果

新增 `CompactionResult`：

```python
@dataclass
class CompactionResult:
    ok: bool
    reason: str          # below_threshold / no_safe_cut_point / summary_failed
                         # / summary_too_long / no_reduction / ok
    before_tokens: int
    after_tokens: int
    summarized_count: int
    kept_count: int
    messages: Optional[List[Dict]]

    @property
    def saved_tokens(self) -> int: ...
```

`compress_with_detail()` 返回它；原 `compress()` 保留为薄封装（`result.messages if result.ok else None`），不动既有调用方。

**新增「压缩无收益则拒绝」**：若 `after_tokens >= before_tokens`，返回 `reason="no_reduction"` 并保留原历史。这是实测倒逼出来的——摘要啰嗦时压缩会让上下文变长。

**渠道路由已接入**：`handler.py` 改用 `compress_with_detail`，日志从「压缩了 N 条」变为「N→M 条，X→Y tokens，省 Z」，且**未压缩时也记录原因**：

```
[history] Compression skipped for session xxx: below_threshold (1200 tokens, 8 messages)
[history] Compressed session xxx: 24 -> 11 messages, 8200 -> 3100 tokens (saved 5100)
```

### 7.3 P3：上下文占用分桶

`request_preview` 新增 `usage` 字段：

```json
"usage": {
  "est_total_tokens": 1594,
  "breakdown": [
    {"source": "system_prompt",       "chars": 2776, "est_tokens": 777, "share": 0.488},
    {"source": "system_tool_schemas", "chars": 0,    "est_tokens": 0,   "share": 0.0},
    {"source": "messages",            "chars": 2882, "est_tokens": 817, "share": 0.512}
  ]
}
```

`source` 取值对齐 ZCode 的 `zcodeContextUsageBreakdownSource`：`system_prompt` / `system_tool_schemas` / `messages`。**现在能直接回答「压哪里最划算」**。

### 7.4 未实施（明确说明）

| 项 | 原因 |
|----|------|
| P2 `mid_turn` 压缩 | 需要改 `loop.py` 工具批次间隙，涉及主循环稳定性，建议单独一轮做 |
| P4 手动 `/compact` | 需要前端入口，属交互改动 |
| 五 trigger + 七 status 状态机 | 为多端 UI 设计，单端场景过度设计 |
| 压缩记录落库 | 依赖 `model_usage` 表扩展，可下一轮加 |

### 7.5 测试增量

| 文件 | 变化 |
|------|------|
| `test_history_compactor.py` | 13 → **22**（+9：token 估算、阈值触发、结构化结果各分支） |
| 全量 | 589 → **650 passed** |

其中一条既有测试因新逻辑失败——它用 `m0`/`m1` 这类单字符消息（32 tokens），压缩后变 37 tokens 被 `no_reduction` 拒绝。**这说明测试数据本身不真实**，已改为有实际体量的内容。真实场景验证：20 条 500 字消息 6820 → 1734 tokens，省 5086，正常压缩。
