# Anthropic Provider 两套流式实现：哪套在用、哪套更好

> 审查时间：2026-10-24
> 对象：`backend/modules/providers/anthropic_provider.py`
> `_chat_stream_via_sdk()` 210 行 / `_chat_stream_via_httpx()` 213 行

---

## 1. 结论先说

| 问题 | 答案 |
|------|------|
| 当前实际在用哪套？ | **两套都不在用**——当前 provider 是 `zhipu`，走的是 `OpenAIProvider` |
| 谁会走 SDK？ | 只有 `provider_id` 为 `None` / `""` / `"anthropic"` 时（`_should_use_sdk()`，行 87-89） |
| 谁会走 httpx？ | `minimax`、`custom_anthropic` 等所有兼容上游（factory 行 73-75） |
| 哪套更好？ | **分场景**：官方 Anthropic 用 SDK 更好；第三方兼容上游必须用 httpx |

---

## 2. 分流机制（已实测验证）

### 2.1 两层分流

**第一层：选 provider 类**（`factory.py:70-78`）

```python
if metadata.id == "anthropic":              provider_class = AnthropicProvider
elif metadata.id in ["minimax", "custom_anthropic"]:  provider_class = AnthropicProvider
else:                                        provider_class = OpenAIProvider
```

**第二层：AnthropicProvider 内部选流式实现**（`anthropic_provider.py:63`）

```python
if self._should_use_sdk():        # provider_id in (None, "", "anthropic")
    → _chat_stream_via_sdk()
else:
    → _chat_stream_via_httpx()
```

### 2.2 实测结果

```
anthropic          -> AnthropicProvider    SDK=True
minimax            -> AnthropicProvider    SDK=False
custom_anthropic   -> AnthropicProvider    SDK=False
zhipu              -> OpenAIProvider       SDK=n/a
```

### 2.3 当前部署的实际状态

```
config.model.provider = "zhipu"
config.model.api_mode = "chat_completions"
```

**当前跑的是 `zhipu`（智谱）→ `OpenAIProvider` → `_chat_stream_via_chat_completions()`。**

也就是说：`anthropic_provider.py` 这个文件**当前一行都没执行**。里面两套实现都是「备而不用」状态。

---

## 3. 为什么这样分流（设计意图）

`_should_use_sdk()` 的注释写得很清楚：`Use the official SDK only for the official Anthropic provider.`

原因很实际：

| 场景 | 用 SDK 的问题 | 用 httpx 的好处 |
|------|--------------|----------------|
| 官方 Anthropic | 无问题，SDK 处理了鉴权头、重试、流式协议细节 | — |
| minimax / 兼容上游 | SDK 会强校验请求结构和响应格式，第三方常有不规范字段 → 直接报错 | 自己解析 SSE，容忍字段缺失/多余 |
| 自建代理/中转 | SDK 可能拒绝非标准 base_url 或 header 组合 | 完全可控 |

这跟 `openai_provider.py` 里那套「兼容性自学习缓存」是同一个思路：**对官方用标准实现，对长尾上游用宽松实现**。

---

## 4. 两套实现的实际差异

### 4.1 共用的部分（8 个核心辅助）

两套都调用同一批方法，说明**能力对等**：

```
_register_content_block_start   # content block 起始登记
_ensure_content_block           # block 懒创建
_finalize_content_blocks        # 结束时的块收尾（thinking + signature 原生往返）
_flush_tool_calls               # 工具调用刷出
_serialize_tool_input           # 参数序列化
_build_usage                    # usage 归一（含 cache_read/cache_write）
_is_timeout_error / _is_auth_error
```

**关键**：`request_params` 在 `chat_stream()` 里**只构建一次**（行 52-59），两条路径共用。所以：
- 上一轮修的「剥离 `reasoning_content` 非规范字段」对两条路径**同时生效**
- `_build_request_params` 里的 cache_control 断点也是共用的

这解释了为什么两套实现没有出现「改一处漏一处」——**请求构建是单点的，只有响应解析分叉**。

### 4.2 各自独有

| 能力 | SDK | httpx |
|------|:---:|:-----:|
| 整流重试（未产出内容前） | ✅ 行 657-671 | ✅ 行 871-891 |
| 已产出内容后超时优雅截断 | ✅ `finish_reason="length"` | ✅ 同 |
| `max_tokens` 超限降级重试 | ❌ | ✅ `_is_missing_max_tokens_error` |
| 手动 SSE 解析 | — | ✅ `_iter_sse_events` |
| 手动状态码处理 | — | ✅ `_raise_for_status` |
| 手动构造 header | — | ✅ `_build_headers` |

**httpx 多出来的三项都是「为了摆脱 SDK 而必须自己做的事」**，不是功能优势。

### 4.3 真正的能力差异

**只有一处**：`max_tokens` 超限降级重试。

某些兼容上游对 `max_tokens` 上限比官方低（比如官方支持 8192、上游只支持 4096），SDK 路径会直接失败，httpx 路径会自动降级重试。

但注意：`_should_use_sdk()` 已经把官方和第三方分开了——官方不会遇到这个问题，第三方本来就走 httpx。**所以这个差异在实际分流下不会造成问题**。

---

## 5. 哪套更好

### 5.1 各自的最优场景

**官方 Anthropic → SDK 更好**

- 鉴权头、`anthropic-version`、beta header 由 SDK 维护，官方一改协议 SDK 跟着更新
- 流式事件类型有类型定义，不会因为上游加字段而解析失败
- 连接池、超时、重试策略是官方调优过的
- **维护成本低**：协议演进由 SDK 承担

**第三方兼容上游 → httpx 更好**

- 不校验响应结构，字段缺失/多余都能容忍
- 可以绕过 SDK 的参数强校验（兼容上游常有非标准写法）
- 可以针对具体上游做特殊处理（`_is_missing_max_tokens_error` 就是这类）

### 5.2 当前设计是对的

分流判断 `provider_id in (None, "", "anthropic")` 恰好落在正确边界上：
- 官方 → SDK（享受官方维护）
- 第三方 → httpx（宽容解析）

**这不是冗余，是有意的双实现。** 我上一轮报告把它列为「结构性问题」，判断偏严——它确实有两份代码，但分工明确、请求构建单点、共用 8 个辅助，重复度远低于 `execute`/`execute_outcome` 那种「同一件事写两遍」。

---

## 6. 真正的风险点（值得改的）

虽然设计合理，但有两个实际隐患：

### 6.1 SDK 未安装时的静默降级

```python
try:
    from anthropic import AsyncAnthropic
except ModuleNotFoundError:
    logger.warning("anthropic package not installed; falling back to raw HTTP client")
```

装了 `anthropic` 包（当前 1.15.0）就走 SDK，没装就静默降到 httpx。**行为不一致但只打一条 warning**——如果官方 API 在 httpx 路径上有细微 bug，用户很难察觉自己在走另一条路。

**建议**：启动时探测一次并记录到日志/设置页，让「当前走哪条路径」可见。

### 6.2 两条路径的行为漂移无人守护

目前没有任何测试断言「两条路径对同一份输入产出等价的 StreamChunk 序列」。

上一轮修的 `reasoning_content` 泄漏问题，如果当时只在 httpx 路径修，SDK 路径就会漏——只是因为 `request_params` 共用才侥幸同时对。

**建议**：加一条契约测试，用同一份模拟 SSE 事件流分别喂给两条路径，断言产出的 `StreamChunk` 序列在关键字段（content/tool_calls/reasoning/usage）上一致。这次不实现，因为需要 mock 掉 SDK 的 `client.messages.create`，工作量大于当前收益。

---

## 7. 可选的优化（不建议现在做）

**能否合并成一套？** 理论上可以：让 httpx 路径作为唯一实现，SDK 只用来「生成请求」不用来「解析响应」。但代价是：

- 放弃 SDk 对协议演进的自动跟进
- 官方 API 的 beta 特性（如新的 thinking 格式）要自己跟
- 当前两条路径都稳定，合并的收益主要是少 200 行代码

**结论：不建议合并。** 这是「用 400 行代码换两类上游都能跑」的合理交易，而项目面对的上游长尾正是它存在的理由。

---

## 8. 一句话回答

**当前一套都没在用**（跑的是 zhipu/OpenAIProvider）。分流规则是：官方 anthropic 走 SDK，minimax/兼容上游走 httpx。

**哪套更好取决于场景**：官方用 SDK（协议演进由官方维护）、第三方用 httpx（容忍不规范响应）。当前分流指针落在正确位置，双实现不是冗余而是分工。

**真正该修的不是「合并两套」，而是**：让「当前走哪条路径」可见，并加一条契约测试守住两条路径的行为一致性。
