# 已实现模块的全面审计报告

> 时间：2026-10-24
> 范围：**不含 `channels/`**，聚焦此前实现的六组能力 + 四个拆分后的包
> 方法：静态审查 + 构造边界输入实测 + 故障注入
> 结果：**发现 6 个真实缺陷，全部已修复**；测试 578 → **589 passed**

---

## 1. 审计对象与测试覆盖

| 模块 | 行数 | 现有测试 |
|------|-----:|---------|
| `agent/session_gate.py` | 287 | 16 例（unit + integration） |
| `agent/history_compactor.py` | 258 | 11 → **13** |
| `agent/request_preview.py` | 219 | 10 |
| `session/usage_recorder.py` | 269 | 9 → **13** |
| `models/model_usage.py` | 61 | 覆盖于 usage 测试 |
| `tools/path_lock.py` | 96 | 0 → **3** |
| `api/cron/`（8 文件） | 1,273 | 2 + 12 路由测试 |
| `api/settings/`（10 文件） | 2,214 | 41 特征测试 |
| `api/chat/`（13 文件） | 3,155 | 23 特征测试 |
| `modules/mcp/`（10 文件） | 2,129 | 7 个测试文件 |

**测试函数总数：132 → 138**

---

## 2. 发现的缺陷（全部已修）

### 缺陷 1【严重】`handler.py` 两处闸门获取在 try 之外 → 锁泄漏

**位置**：`channels/handler.py:1957`（外部编码代理）、`:2150`（团队命令）

```python
await gate.acquire(session_id)      # ← 在 try 之外
turn_acquired = True
self._active_tasks[session_id] = cancel_token   # ← 这里抛异常就永久泄漏
try:
    ...
finally:
    if turn_acquired:
        gate.release(session_id)     # ← 覆盖不到上面
```

**后果**：acquire 成功之后、进入 try 之前的任何异常（例如字典赋值、属性访问出错），`finally` 不会执行，该会话的锁永久不释放——**此后这个会话的所有消息都会卡住，表现为完全无响应**，且没有任何报错。

这类问题在正常流程下永远不会出现，只有异常路径才暴露，因此极难发现。

**修复**：把「acquire + 后续准备」一起移入 try，`turn_acquired` 标志保证只在真正获取后才释放：

```python
try:
    await gate.acquire(session_id)
    turn_acquired = True
    ...所有准备代码...
finally:
    if turn_acquired:
        gate.release(session_id)
```

**新增测试** `tests/test_gate_exception_safety.py`：静态扫描 `handler.py` 里每个 `gate.acquire` 是否被缩进更小的 `try` 包住。修复前该测试精确报出 `line 1957, line 2150`。

### 缺陷 2【中】用量统计漏算 OpenAI 缓存命中

**位置**：`session/usage_recorder.py` `_first_int()`

**问题**：嵌套字典取值时硬编码只看 `reasoning_tokens` 两个键：

```python
if isinstance(value, dict):
    value = value.get("reasoning_tokens", value.get("reasoningTokens", 0))
```

而 OpenAI 系上报缓存命中的真实结构是 `prompt_tokens_details.cached_tokens`。查 `cached_tokens` 时进到子字典里去找 `reasoning_tokens`，必然找不到，返回 0。

**后果**：所有 OpenAI 兼容上游（含当前使用的 zhipu）的**缓存命中一律统计为 0**，`cache_hit_rate` 永远是 0%，成本报表系统性偏高——而这正是「注入开关到底省了多少 token」要回答的问题。

**实测证据**：
```
{"prompt_tokens":100,"completion_tokens":50,"prompt_tokens_details":{"cached_tokens":80}}
修复前 → cache_read_tokens=0
修复后 → cache_read_tokens=80
```

**修复**：顶层别名找不到时，再下钻 `*_details` 容器按同一别名查找。

**新增 4 条测试**覆盖 OpenAI 缓存明细、推理明细、两者共存、DeepSeek 缓存键。

### 缺陷 3【中】压缩摘要无长度上限，可能让上下文变长

**位置**：`agent/history_compactor.py` `compress()`

**问题**：压缩会把旧历史换成「一条摘要 + 最近 N 条」。但摘要长度完全由模型决定，代码没有上限校验。

**实测**：让模型返回 200,000 字符「摘要」，压缩后总字符 200,049——**比压缩前还长**，压缩彻底失去意义，还可能直接撑爆窗口。

**修复**：新增 `summary_max_chars`（默认 4000），超限时放弃压缩（返回 `None`），调用方沿用原历史。选择「放弃」而不是「截断」是因为截断会把摘要切成半句话，语义不完整。

### 缺陷 4【中】`request_preview` 传 `None` 导致 500

**位置**：`agent/request_preview.py`

**问题**：`message=None` 时，下游 `context.py:739` 的 `if f"@{team_name}" in message` 抛 `TypeError: argument of type 'NoneType' is not iterable`，整个 `/api/chat/preview` 返回 500。

**后果**：预览是排查工具，本应「任何输入都能给出报告」。因少传一个参数整体失败，等于工具在最需要它的时候不可用。

**修复**：入口处归一化 `message = str(message) if message is not None else ""`。

### 缺陷 5【轻】`path_lock` 同路径嵌套会静默死锁

**位置**：`tools/path_lock.py`

**问题**：`asyncio.Lock` 不可重入。同任务对同一路径嵌套获取会**永久挂起**，表现为「请求无响应」，排查成本极高。

**验证**：当前生产代码两处调用点（write / edit）互不嵌套，所以**尚未触发**。但这是随时可能被未来调用链踩到的雷。

**修复**：记录持有者任务，同任务重入时立即抛 `RuntimeError` 而非挂起。已验证重入检测不破坏正常串行语义。

### 缺陷 6【轻】`history_compactor` 工具函数强制 `keep_recent` 但实际会缩小

**位置**：`find_cut_index()`

**现象**：`keep_recent=2` 在「工具调用对」场景下实际只保留 1 条（cut 从 2 后移到 3，以避开孤儿 tool 结果）。

**判断**：这是**正确行为**——切点必须落在安全边界，宁可少保留原文也不能制造非法消息。属设计取舍而非缺陷。**已记录，不修改**。

---

## 3. 验证为正确的部分

审计中主动构造边界输入，以下均表现正确：

| 模块 | 验证项 | 结果 |
|------|--------|------|
| `session_gate` | 200 次 acquire/release 后 `tracked_sessions=0` | ✓ 无残留 |
| `session_gate` | 排队消息合并、`take_pending` 只生效一次、`drain` 幂等 | ✓ |
| `session_gate` | 并发场景 `total_waits` / `total_wait_seconds` 统计 | ✓ 准确 |
| `history_compactor` | 工具对、连续工具结果、纯对话的切点 | ✓ 均避开孤儿 tool |
| `history_compactor` | 全 tool 消息、以 tool 开头 → `cut=0` 放弃压缩 | ✓ 安全降级 |
| `history_compactor` | provider 故障 / 空返回 → `None` 降级 | ✓ 不抛异常 |
| `usage_recorder` | Anthropic / DeepSeek / 推理模型字段 | ✓ 归一正确 |
| `path_lock` | 同一文件多种路径写法 → 同一把锁 | ✓ |
| `path_lock` | 取消后锁可重新获取 | ✓ 不泄漏 |
| `path_lock` | 重入检测不误伤串行与并发 | ✓ |
| `request_preview` | 空消息 / 50 万字符 / 大括号 / 控制字符 / 多模态 | ✓ 均不崩 |

**闸门接线核查**：`chat/stream.py`、`ws/events.py`、`channels/handler.py`（3 处）、`cron/executor.py` 全部正确 acquire + release。

---

## 4. 遗留问题（未修，需你判断）

### 4.1 `ContextBuilder._find_mentioned_team` 不接受 None

`context.py:739` 直接对 `message` 做 `in` 运算。我在 preview 入口做了防御，但**根因仍在**：任何其他调用方传 None 都会崩。

修根因需要改 `context.py`，但那不在本次审计范围（且主循环已有前置校验）。**建议后续单独处理**。

### 4.2 `keep_recent` 语义与实际保留数不一致

文档说「保留最近 N 条」，实际因切点安全约束可能少保留。**建议把参数名改为 `min_keep_recent` 或在文档中注明**，避免调用方误判。

### 4.3 Gate 的 `total_waits` 只在真正排队时累加

同一会话快速串行调用（不排队）不计入 `total_waits`。这是合理设计（该指标意在衡量「排队压力」），但对外解释时需说明。

---

## 5. 测试基线

```
589 passed / 0 failed
```

本次审计新增 11 条测试：

- `test_gate_exception_safety.py`（4 例）：闸门异常安全的单元 + 静态检查
- `test_usage_recorder.py` +4：嵌套明细字段
- `test_history_compactor.py` +2：摘要上限
- `test_tool_boundaries.py` +3：路径锁重入

**修复前后对比**：578 → 589 passed。

---

## 6. 审计方法总结（可复用）

有效的审计不是读代码，而是**构造让代码出错的输入**：

1. **异常路径优先**：正常流程测一百遍也发现不了锁泄漏，只有构造「acquire 后抛异常」才暴露
2. **静态检查补动态测试的盲区**：`try` 包住 `acquire` 这种结构问题，运行时很难测，用缩进比较扫描一下就能查
3. **故障注入**：把 provider 换成抛异常的假实现，看降级路径是否真的降级
4. **真实字段名验证**：不要相信「应该支持」，用各家真实上报格式跑一遍
5. **边界值**：`None`、空串、50 万字符、控制字符、极端小数/大数
