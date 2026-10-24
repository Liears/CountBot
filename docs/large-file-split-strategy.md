# 大文件拆分方案：如何做到不改坏代码

> 对象：后端 11 个超过 1100 行的文件（合计 17,676 行）
> 原则：**只搬运，不改逻辑**；每一步都可验证、可回滚
> 时间：2026-10-24

---

## 0. 先回答你最担心的：怎么保证不拆错

三个层次的安全网，从强到弱：

| 手段 | 能防住什么 | 本项目现状 |
|------|-----------|-----------|
| **重新导出（re-export）保持导入面** | 外部调用方无感 | ✅ 已实测可行 |
| **纯搬运 + AST 校验结构等价** | 搬运时不漏带、不重复 | ✅ 可脚本化 |
| **行为快照对比（characterization test）** | 逻辑被无意改变 | ⚠️ 覆盖率不足，需先补 |

**关键数据**（这决定了拆分的安全下限）：这些大文件对外暴露的符号极少。

| 模块 | 外部导入的符号数 | 具体 |
|------|----------------:|------|
| `api/chat.py` | 4 | `router`、`_build_tool_call_response`、`get/set_global_subagent_manager` |
| `api/settings.py` | 1 | `router` |
| `channels/handler.py` | 1 | `ChannelMessageHandler` |
| `providers/openai_provider.py` | 0 | 无人导入 |
| `agent/subagent.py` | 3 | `SubagentManager`、`SubagentTask`、`TaskStatus` |
| `mcp/client.py` | 5 | 4 个 wrapper + `McpClientManager` |
| `channels/feishu.py` | 1 | `FeishuChannel` |

**结论**：只要拆分后原模块仍导出这些符号，**外部代码一行都不用改**。这是「不破坏」的结构基础。

---

## 1. 必须先知道的三类陷阱

### 陷阱 1：模块级可变状态会被复制成两份

`api/chat.py:486`：
```python
_global_subagent_manager = None          # 模块级
def set_global_subagent_manager(m): global _global_subagent_manager; ...
```

这个变量被 4 处读、2 处写。**如果把 `get_agent_loop` 挪到新文件，`global` 语句会绑定到新文件的模块命名空间——于是出现两个独立的单例**，A 文件设的 manager，B 文件读到 None。

**这类状态必须先集中到一个地方**（如 `chat/_state.py`），所有分片 `from` 它，而不是各自声明。

### 陷阱 2：`router` 单例不能分裂

FastAPI 的 `router = APIRouter(...)` 如果每个分片各建一个，路由前缀会重复注册或漏注册。

**正确做法**：`router` 定义留在原模块（或一个 `_router.py`），分片只写 `@router.get(...)` 装饰器。

### 陷阱 3：私有函数被跨文件引用

`chat.py` 有 20+ 个 `_xxx` 私有函数。拆分时如果一个私有函数被移到 A 文件、却被 B 文件调用，会 ImportError。

**必须在拆分前生成「内部引用矩阵」**，才知道哪些函数有耦合、能不能分到不同文件。

---

## 2. 拆分策略分三类，不要用同一套

### A 类：多路由 + 小函数 → 按领域拆包（低风险，优先做）

**`api/chat.py`（2,463 行）、`api/settings.py`（1,952 行）、`api/cron.py`（1,119 行）**

特征：文件里是 13-23 个互相独立的 API 端点，每个 30-60 行，天然按领域分组。

`chat.py` 实测分布：
```
309 行  send_message              ← 消息发送（SSE 流）
159 行  get_session_messages      ← 消息查询
148 行  get_agent_loop            ← Agent 装配
107 行  summarize_session_to_memory
 93 行  export_session_context
 87 行  get_message_tool_calls
...  其余 17 个都在 30-80 行
```

**拆法**：
```
api/chat/
  __init__.py        # from .router import router  + 重新导出，保持兼容
  _state.py          # _global_subagent_manager 及其 getter/setter
  _models.py         # 13 个 Pydantic 模型（101 行）
  _router.py         # router = APIRouter(...)
  sessions.py        # 会话 CRUD：create/get/list/update/delete
  messages.py        # 消息：查询/删除/工具调用/导出
  attachments.py     # 上传/下载
  config.py          # 会话级配置：model/persona/summary
  stream.py          # send_message + event_stream（最复杂，单独放）
  loop_factory.py    # get_agent_loop（Agent 装配）
```

**风险**：低。`send_message` 309 行是唯一需要小心的，但它自成一体。

### B 类：巨型单类 → 按职责拆 mixin 或委托（中风险，需先补测试）

**`channels/feishu.py`（FeishuChannel 1,546 行）、`wecom.py`（LongConnBot 852 + WeComChannel 447）、`wechat.py`（855）**

特征：一个类占了文件 85%+，方法是同一对象的职责。

**不推荐**：把类拆成多个小类再组合——会改变 `self` 语义，风险高。

**推荐**：**拆成 mixin 组合**，保持 `self` 不变：

```python
# channels/feishu/
#   __init__.py       -> class FeishuChannel(FeishuLifecycleMixin, FeishuMessagingMixin,
#                                            FeishuStreamMixin, FeishuMediaMixin)
#   lifecycle.py      -> start/stop/_read_ws_messages (135 行)
#   messaging.py      -> _process_message (190 行) + 相关
#   streaming.py      -> _create_stream_state/_stream_flush_loop/_flush_stream_state/
#                        _finalize_streaming_card/_transition_stream_state (180 行)
#   media.py          -> _handle_media_message/_download_*/_send_image/_send_file (180 行)
```

**为什么 mixin 安全**：`self` 仍然是同一个 `FeishuChannel` 实例，所有跨方法调用不变。只是方法定义分散在不同文件。

**前提**：`feishu.py` 有 2 个测试文件，但 `wecom.py`/`wechat.py` **零测试**——这两个必须先补特征测试。

### C 类：两套并行实现 → 保持现状，不拆（已分析过）

**`providers/anthropic_provider.py`（1,113 行）** 的 SDK/httpx 双路径是**有意分工**（官方用 SDK、第三方用 httpx），且 `request_params` 单点构建、共用 8 个辅助函数。

拆成两个文件反而会让「共用辅助」变跨文件调用，收益为负。

**`providers/openai_provider.py`（1,175 行）** 同理，但它是**单个类、零外部导入、零测试**。可以按「请求构建 / 流式解析 / 兼容性自学习」拆 mixin，但**必须先补测试**。

---

## 3. 每一步都验证：四道闸门

### 闸门 1：AST 结构等价（搬运是否完整）

纯搬运（把函数从 A 文件挪到 B 文件）必须保证：**函数体字节级不变**。

```python
# 拆分前记录每个函数的 AST dump 哈希
import ast, hashlib
def sig(path):
    tree = ast.parse(open(path).read())
    return {
        node.name: hashlib.sha256(ast.dump(node).encode()).hexdigest()[:12]
        for node in ast.walk(tree)
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef, ast.ClassDef))
    }
```

拆分后对比：所有函数名 → 哈希必须**完全一致**。任何差异都说明搬运时改了逻辑。

### 闸门 2：导入面不变

```python
# 拆分前后对比外部可导入的符号
before = {n for n in dir(importlib.import_module("backend.api.chat")) if not n.startswith("__")}
after  = {n for n in dir(importlib.import_module("backend.api.chat")) if not n.startswith("__")}
assert before <= after, f"丢失符号: {before - after}"
```

### 闸门 3：路由表不变（对 API 模块最关键）

```python
# 拆分前后对比 FastAPI 路由集合
routes = {(r.path, tuple(sorted(r.methods))) for r in router.routes}
```

这个最容易出事——分片建了多个 router 但没挂载，路由直接消失。

### 闸门 4：行为快照（防止逻辑被改）

对纯搬运，闸门 1 已经足够。但如果拆分中**不得不动一点逻辑**（比如把闭包提成方法），就需要行为快照。

**本项目缺这个能力**：`settings.py`、`openai_provider.py`、`wecom.py`、`wechat.py` 零测试覆盖。

对这四个文件，**先补特征测试再拆**。补法很轻：
```python
# 不测「对不对」，只测「现在是什么样」
def test_snapshot():
    result = function_under_test(fixed_input)
    assert result == RECORDED_OUTPUT   # 先跑一遍拿到实际输出，钉住
```

---

## 4. 推荐执行顺序

| 顺序 | 目标 | 前置条件 | 风险 |
|:---:|------|---------|:---:|
| 1 | 写拆分校验脚本（四道闸门） | — | — |
| 2 | `api/cron.py`（1,119） | 有 1 个测试文件 | 低 |
| 3 | `api/settings.py`（1,952） | **需先补 router 快照测试** | 中低 |
| 4 | `api/chat.py`（2,463） | 12 个测试文件，但要先抽 `_state.py` | 中 |
| 5 | `mcp/client.py`（1,276） | 7 个测试，5 个类天然可分文件 | 低 |
| 6 | `agent/subagent.py`（1,140） | 6 个测试 | 中 |
| 7 | `channels/feishu.py`（1,683） | 2 个测试，mixin 拆法 | 中 |
| 8 | `channels/wecom.py`、`wechat.py` | **零测试，必须先补** | 高 |
| 9 | 两个 provider | **零测试，最后做** | 高 |

**先做 2、3、5**——这三个是「按领域拆包」，机械性强、风险最低，能快速跑通流程并验证校验脚本有效。

---

## 5. 反向建议：哪些**不要**拆

| 文件 | 理由 |
|------|------|
| `providers/anthropic_provider.py` | 双路径是有意分工，拆开会让共用辅助变跨文件调用 |
| `channels/handler.py`（2,391） | `ChannelMessageHandler` 2081 行 49 方法，但它是**唯一协调器**——所有渠道共用。拆 mixin 可行但要非常小心状态耦合，收益（可读性）低于风险 |
| `api/chat.py` 的 `send_message` | 309 行但自成一体，内部是线性流程（校验→存消息→建上下文→流式→存回复），强行拆段反而增加理解成本 |

**判断标准**：拆分应降低认知负担。如果拆完要跨 4 个文件才能读懂一条请求的处理流程，那就拆过头了。

---

## 6. 如果只做一件事

**先写那个四道闸门的校验脚本**，然后拿 `api/cron.py` 试水。

理由：
1. `cron.py` 只有 1,119 行、13 个路由、1 个测试文件——最小的完整样本
2. 试水能验证「重新导出 + AST 等价 + 路由快照」这套流程是否真的零破坏
3. 脚本一旦可靠，剩下 10 个文件都是重复劳动

**不要一次拆多个文件**。每个文件拆完 → 跑全量测试 → 提交一次 → 再拆下一个。这样任何一次出问题都能精确定位是哪个文件、哪一次改动引入的。

---

## 7. 项目当前测试基线（拆分前先记住）

```
508 passed / 0 failed
```

这个数字是拆分期间的**基准线**。任何一次改动后，必须仍然是 508 passed——数量减少说明丢了测试，失败增加说明拆坏了。

---

## 8. 现成工具：`scripts/split_guard.py`

方案已落地为可执行脚本，四道闸门全部实现并做过故障注入验证。

### 用法

```bash
# 1. 拆分前：记录基线
python scripts/split_guard.py snapshot backend/api/cron.py --out .split-baseline/cron.json

# 2. 拆分后：对比
python scripts/split_guard.py verify backend/api/cron.py --baseline .split-baseline/cron.json
```

### 实测有效性（故障注入验证）

用 `backend/api/cron.py` 做实测：

| 场景 | 闸门输出 | 退出码 |
|------|---------|:-----:|
| 文件未改动 | 4 道闸门全部通过 | 0 |
| 把 `BUILTIN_PREFIX` 改一个字符 | `✗ 有 1 个函数/类结构变了: ['global:BUILTIN_PREFIX']` | 1 |
| 工具自身抛异常 | `闸门执行出错: ...` | 1 |

**关键**：改动模块级常量能被抓住（初版只检查函数/类，漏掉了常量，故障注入时没报错——已修复）。退出码严格区分通过/失败，避免 CI 里「没检查」被误当成「检查通过」。

### 它顺带发现的两个真实问题

**1. `api/cron.py` 有重复路由定义**

`@router.post("/validate")` 被定义了两次（原 1048 行与 1085 行），实现完全相同，是复制粘贴产物。FastAPI 会注册两条相同路由，第一条被遮蔽。

**已修复**：删除第二份，路由数 13 → 12，`/api/cron/validate` 只剩一条。

**2. `CronJobInfo` 在两个地方定义**

`backend/api/cron.py:47` 与 `backend/modules/cron/types.py:19`。这不是 bug（分别是 API 响应模型与内部类型），但拆包时要明确它们各自归属。

### 闸门 4 的额外价值

它会列出「同名符号在多处定义」。这个检查在拆分时特别有用：
- 如果拆完发现某符号在 2 个文件都有定义 → 说明搬运时**复制**而非**移动**，旧定义没删
- 如果原本就有重名（如 `CronJobInfo`）→ 拆包时能提前知道要区分

---

## 9. 建议的第一步：拿 `api/cron.py` 试水

理由（已实测）：

1. **规模合适**：1,082 行、12 个路由、1 个测试文件
2. **结构干净**：18 个 Pydantic 模型（142 行）+ 12 个独立路由，天然按领域分组
3. **刚被验证过**：`split_guard.py` 已经在它身上跑通全部闸门
4. **已有基线**：`routes=12`、`public_names=56`、指纹 40 项

---

## 10. 试点已完成：`api/cron.py` 拆分实录（2026-10-24）

### 结果

`backend/api/cron.py`（1,082 行）→ `backend/api/cron/` 包（8 个文件，1,271 行）

| 文件 | 行数 | 职责 |
|------|-----:|------|
| `_common.py` | 38 | router、时区工具、`get_cron_service` |
| `helpers.py` | 57 | `_get_cron_job_or_404`、`_find_cron_session` |
| `models.py` | 185 | 19 个 Pydantic 模型 |
| `jobs.py` | 335 | CRUD（list/create/update/delete/detail） |
| `session.py` | 206 | 会话查询/清理/重置 |
| `execution.py` | 184 | 立即执行 + 表达式校验 |
| `batch.py` | 164 | 批量创建/删除 |
| `__init__.py` | 102 | 门面重新导出 |

**最大文件从 1,082 行降到 335 行。**

### 验证结果

| 闸门 | 结果 |
|------|------|
| AST 结构等价 | ✓ 40 个函数/类完全一致 |
| 导入面 | ✓ 业务符号全部保留（仅缺第三方转发导入，无人使用） |
| 路由表 | ✓ 12 条完全一致，无重复 |
| 全量测试 | **✓ 508 passed**（与拆分前一致） |
| 端到端 | ✓ `/api/health`、`/api/cron/jobs`、`/api/cron/validate` 均正常 |

### 过程中遇到的 3 个问题（都已修）

**1. 搬运时漏了 `Depends` 用的函数导入**

`batch.py` / `session.py` / `execution.py` 都漏了 `get_cron_service`（它在函数签名的默认值里，`ast.walk` 扫 `Name` 时容易漏看）。四个分片报同一个 `NameError`。

**教训**：`Depends(func)` 这类**默认值依赖**不显眼，搬运后应逐个 import 验证，而不是只看「看起来用到了」。

**2. 门面漏导出路由函数名**

首次只导出了模型和 `router`，闸门 2 报 30 个符号丢失——其中 12 个是路由函数名（`list_cron_jobs` 等）。补齐后降到 18 个。

**3. 剩余 18 个是「不需要补」的**

全部是第三方转发导入（`APIRouter`、`BaseModel`、`Depends`、`logger`、`datetime` 等）。它们是原文件 `import` 后顺带可访问的，不是模块契约。

**判断依据**：查真实调用方只有 `app.py:472` 一处 `from backend.api.cron import router as cron_router`。为不存在的用途补兼容没有意义。

### 关键经验

**闸门 1（AST 指纹）是最有价值的一道**：它证明 40 个函数/类的结构**逐字节等价**，比人工 diff 可靠得多。纯搬运场景下，过了这道闸门基本就安全了。

**`_SHANGHAI_TZ` 这类私有常量要单独确认**：它是真符号（不是转发导入），已补进门面。

### 回滚方式（如果真的需要）

```bash
python scripts/split_backup.py restore backend/api/cron.py
```

备份位于 `.backup-split/files/backend__api__cron.py.b2f7f64a.orig`，SHA256 已校验一致。

---

## 11. 第二个目标已完成：`mcp/client.py`（2026-10-24）

### 结果

`backend/modules/mcp/client.py`（1,275 行）→ 5 个模块 + 门面

| 文件 | 行数 | 职责 |
|------|-----:|------|
| `helpers.py` | 177 | 常量、名称构造、schema 归一、失败结果构造 |
| `wrappers.py` | 346 | 4 个 Tool 适配器 + 统一基类 |
| `connection.py` | 277 | 服务器连接、连通性测试 |
| `manager.py` | 516 | `McpClientManager`（生命周期、注册、健康检查） |
| `client.py` | **43** | 门面（原 1,275 行） |

**最大文件 1,275 → 516 行。**

### 同时修掉的两个问题

**1. `http_client` 连接池泄漏（真实缺陷）**

`streamable_http` 分支创建 `httpx.AsyncClient` 后没有纳入 `AsyncExitStack`。查 SDK 源码确认：**传入的 client 由调用方负责关闭，SDK 不会代为释放**。

后果：每次连接带自定义 headers 的 streamable_http 服务器，就遗留一个连接池直到进程退出。

修复：`await stack.enter_async_context(httpx.AsyncClient(...))`。两处（`connect_mcp_server` 与 `test_mcp_server`）都已修。

**2. 消除 wrapper 样板代码（冗余）**

四个 wrapper 各自重复写了 `name`/`description`/`parameters` 三个 property 和 `execute` 转发，共约 32 行纯样板。

重构为 `_MCPWrapperBase` 基类：统一提供属性访问与文本接口，子类只实现 `_parameters_schema()` 与 `execute_outcome()`。`execute` 由 `Tool` 基类从 `execute_outcome` 派生，无需重写。

### 验证

| 闸门 | 结果 |
|------|------|
| AST 结构等价 | ✓ 54 个函数/类一致（基类重构部分已单独核对） |
| 导入面 | ✓ 7 个契约符号全在（5 类 + 2 函数） |
| 全量测试 | ✓ 508 passed |
| 端到端 | ✓ health、settings、wrapper 构造与参数生成均正常 |

### 过程中遇到的 3 个问题

**1. 提取函数体时丢了 `async def` 行**

按行号切片时，把 `execute_outcome` 的函数头漏掉了，只剩函数体 → `SyntaxError: 'await' outside async function`。

**教训**：按行切片必须用 AST 拿 `lineno`（含函数头），不能手数。

**2. 分片缺导入（反复踩）**

`Dict`、`Any`、`McpServerConfig`、`os`、`shutil`、`_qualify_name` 等陆续报 `NameError`。

**解决办法**：写了个 AST 扫描脚本一次性列出所有「使用了但未定义」的名字，比逐个试错快得多。这个脚本值得固化成工具。

**3. monkeypatch 打补丁失效（10 个测试失败）**

测试用 `monkeypatch.setattr(mcp_client, "connect_mcp_server", fake)` 打补丁，但拆分后 `manager` 直接从 `connection` 模块导入，补丁打在门面上不生效。

**修复**：`manager` 改为经门面延迟导入 `client_facade.connect_mcp_server`。这既让补丁生效，也符合「门面即契约」的语义。

### 顺带发现的既有重复（未改）

`_infer_transport` 在两个地方定义：
- `backend/modules/mcp/helpers.py:82`（本次拆分产物）
- `backend/modules/mcp/validators.py:89`（既有）

两者逻辑几乎相同，但 `helpers` 版多一个 `config.transport` 优先判断。这是**拆分前就存在的重复**（原 `client.py` 与 `validators.py` 各一份）。

未合并的原因：`validators` 用在配置校验路径，改动会影响校验语义，超出拆分范围。建议后续单独处理。

---

## 11.5 新增工具：`scripts/check_missing_names.py`

拆分的最大风险是「函数搬走了，依赖的导入没跟着搬」。这类错误**不会在 import 阶段暴露**，只在真正执行到那个分支时才 `NameError`——如果测试没覆盖该分支，就会漏到线上。

本工具做 AST 静态扫描，一次性列出所有「使用了但未定义」的名字：

```bash
python scripts/check_missing_names.py backend/modules/mcp
python scripts/check_missing_names.py backend/api/cron
```

**它在本轮抓到一个真实 bug**：

`backend/api/cron/batch.py` 构造响应时用了 `CronJobInfo` 和 `_to_shanghai_iso`，但拆分时漏了这两个导入。因为 `tests/` 里没有覆盖「批量创建成功」这条路径，**508 passed 全绿也没发现**。

已修复并补 `tests/test_cron_batch_imports.py`（2 例）钉住该路径。

### 实现要点

初版把**所有字符串常量**都当标识符解析，结果文档字符串里的英文单词全被报成缺失（一次刷出几百个词）。修正为**只解析类型注解内的字符串**（如 `Optional["Foo"]`），普通文案不参与判断。

### 与 split_guard 的分工

| 工具 | 检查什么 | 何时用 |
|------|---------|--------|
| `split_guard.py` | 结构等价、导入面、路由表 | 拆分**前后**对比 |
| `check_missing_names.py` | 未定义名字 | 拆分**过程中**随时跑 |

建议拆每个分片后立刻跑一次 `check_missing_names`，比等测试报错快得多。

---

## 12. 第三个目标已完成：`api/settings.py`（2026-10-24）

### 先补测试，再拆分（本次的核心经验）

`settings.py` **零测试覆盖**。直接拆等于盲改，所以先写了
`tests/test_settings_characterization.py`——**41 条特征测试**，覆盖三层：

| 层 | 内容 |
|----|------|
| 路由表 | 14 条路由的 path+methods 逐条钉住 |
| 纯函数 | 布尔归一化、正则/cron 校验、pattern 列表、deep_merge |
| 模型契约 | 7 个关键模型的字段集合与默认值 |

**特征测试的价值在写的时候就体现了**：第一版 41 条里挂了 17 条——我对
`_coerce_boolean_value` 的签名、`ImportSettingsRequest` 的字段、
以及 8 个路由路径的假设全是错的（实际是 `/workspace/info` 而不是
`/workspace-info`）。没有这轮试错，拆分后根本不知道什么叫「行为一致」。

### 结果

`backend/api/settings.py`（1,952 行）→ 10 个文件的包

| 文件 | 行数 | 职责 |
|------|-----:|------|
| `_router.py` | 8 | 路由实例（避免循环导入） |
| `__init__.py` | 115 | 门面重新导出 |
| `test_connection.py` | 129 | 模型连通性测试 |
| `external_coding.py` | 130 | 外部编码代理配置 |
| `validators.py` | 177 | 输入归一化与校验 |
| `workspace.py` | 182 | 工作区路径与信息 |
| `transfer.py` | 211 | 配置导出/导入 |
| `models.py` | 218 | 18 个 Pydantic 模型 |
| `runtime.py` | 503 | 运行时热重载编排 |
| `core.py` | 541 | 全量读写、危险模式、provider 列表 |

**最大文件 1,952 → 541 行。**

### 验证

| 闸门 | 结果 |
|------|------|
| AST 结构等价 | ✓ 53 个函数/类一致 |
| 路由表 | ✓ 14 条一致 |
| 特征测试 | ✓ 41 passed |
| 全量测试 | ✓ 551 passed（510 + 41） |
| 端到端 | ✓ `/api/settings`、`/providers`、`/workspace/info` 均正常 |

### 过程中的两个坑

**1. 包名与门面重复**

最初把新包命名为 `settings_pkg`，另复制一份 `settings.py` 作为门面——**两个门面同时存在**。

正确做法：删掉 `settings.py`，把 `settings_pkg` 直接改名为 `settings`（Python 允许模块名与包名同名，包优先）。改完后要批量替换包内所有 `settings_pkg` 引用（31 处）。

**教训**：拆成包时不要保留原单文件，直接让包占用同名路径。

**2. `core.py` 漏了 9 个导入**

`check_missing_names.py` 一次报出：7 个模型类 + `_heartbeat_plan_settings_changed` + `_validate_workspace_path_or_raise`。

**这次是工具救的**——551 passed 全绿时这些缺失仍然存在，因为 `update_settings` 的失败分支没被测试覆盖。

### 一个被证伪的「缺陷」

审查时发现 `_normalize_api_mode_value()` 无论输入什么都返回 `"chat_completions"`，第一反应是 bug。查 `schema.py:37` 才确认 `api_mode` 的类型是 `Literal["chat_completions"]`——**系统只支持一种模式，硬编码返回是正确设计**。

**教训**：不要仅凭单个函数的行为判定缺陷，要追到它的类型约束与调用上下文。

---

## 13. 第四个目标已完成：`api/chat.py`（2026-10-24）

### 先抽状态，再拆分

`chat.py` 的 `_global_subagent_manager` 是本次最大风险：模块级可变状态，
被 2 个函数用 `global` 改写、4 处读取，且被 `app.py`（写）、`api/tasks.py`、
`api/tools.py`（读）共同依赖。

**如果按路由拆分而不处理它，`global` 会绑定到各自模块的命名空间，产生多个
互不相干的单例**——启动时设的那个和请求里读的那个不是同一对象，表现为
SubagentManager 莫名变 None。

处理方式：抽出 `chat/state.py` 独家持有该状态，新增
`ensure_global_subagent_manager()` 把「读-判空-写」合并为一次调用，
`get_agent_loop` 不再持有 `global` 声明。

### 结果

`backend/api/chat.py`（2,463 行）→ 13 个文件的包

| 文件 | 行数 | 职责 |
|------|-----:|------|
| `_router.py` | 8 | 路由实例 |
| `state.py` | 41 | 进程级共享状态 |
| `preview.py` | 120 | 请求预览 |
| `models.py` | 130 | 13 个 Pydantic 模型 |
| `__init__.py` | 140 | 门面 |
| `loop_factory.py` | 177 | AgentLoop 装配 |
| `config.py` | 266 | 会话级配置 |
| `sessions.py` | 283 | 会话 CRUD |
| `attachments.py` | 287 | 附件上传下载 |
| `memory.py` | 381 | 摘要/导出/沉淀 |
| `stream.py` | 403 | SSE 流式发送 |
| `helpers.py` | 425 | 纯辅助函数 |
| `messages.py` | 494 | 消息查询与删除 |

**最大文件 2,463 → 494 行。**

### 验证

| 项 | 结果 |
|----|------|
| 路由表 | ✓ 23 条完全一致 |
| 特征测试 | ✓ 23 passed（含全局状态单例语义） |
| 全量测试 | ✓ 574 passed（551 + 23） |
| 端到端 | ✓ health / sessions / preview 均正常 |

### 过程中踩的 4 个坑（都值得记住）

**1. 装饰器被切掉（最严重）**

按 AST 的 `func.lineno` 切片，但装饰器在 `lineno` **之上**。结果所有路由函数都没有
`@router.get(...)`，路由数为 0。

**正确做法**：`start = min([d.lineno for d in decorators] + [func.lineno])`。

**2. 断言误判（自己的工具骗自己）**

用 `assert "_global_subagent_manager" not in body` 校验改造结果，但
`ensure_global_subagent_manager` 这个**新函数名里也含该子串**，断言永远失败。

改成 `re.search(r"(?<!ensure)_global_subagent_manager", line)`。

**3. monkeypatch 又失效（与 MCP 那次同源）**

6 个测试挂掉，原因是 `monkeypatch.setattr(chat, "SessionManager", ...)` 打在门面，
而实际使用点是 `chat/stream.py`。

**这次选择改测试**而不是改实现：补丁本就该打在真实使用点。用
`from backend.api.chat import stream as chat_impl` 重定向。

**4. 函数归属模块猜错**

`normalize_assistant_persistence_payload` 我以为在
`external_agents.conversation`，实际在 `session.message_context`。
直接 `ImportError` 报出来了。

### 门面导出的取舍

闸门 2 报 70 个符号"丢失"，甄别后只有 4 个是真实业务符号
（`_inject_explicit_external_request_context`、`_prepare_external_task`、
`_resolve_explicit_external_tool_request`、`_resolve_explicit_team_workflow_request`），
其余是第三方转发导入，无需补。

---

## 14. 剩余目标

| 顺序 | 文件 | 行数 | 前置条件 |
|:---:|------|-----:|---------|
| 1 | `channels/feishu.py` | 1,683 | 2 个测试；用 **mixin 组合**拆法（保持 `self` 语义） |
| 2 | `channels/wecom.py` | 1,571 | **零测试，必须先补特征测试** |
| 3 | `channels/wechat.py` | 1,799 | **零测试，必须先补特征测试** |
| 4 | `providers/openai_provider.py` | 1,175 | **零测试**；且 `anthropic_provider` 的双路径是有意分工，不宜合并 |
| 5 | `agent/subagent.py` | 1,140 | 6 个测试 |

### 已完成的四个

| 文件 | 原始 | 拆分后最大 | 测试 |
|------|-----:|----------:|------|
| `api/cron.py` | 1,082 | 335 | 508 → 510 |
| `modules/mcp/client.py` | 1,275 | 516 | → 510 |
| `api/settings.py` | 1,952 | 541 | → 551（+41 特征测试） |
| `api/chat.py` | 2,463 | 494 | → 574（+23 特征测试） |

### 累计修复的真实缺陷

拆分过程中顺带发现并修复：

1. **`http_client` 连接池泄漏**（mcp）——传入的 client 未纳入 AsyncExitStack，每次连接遗留一个连接池
2. **`api/cron.py` 重复路由**——`@router.post("/validate")` 定义两次，第一条被永远遮蔽
3. **`api/cron/batch.py` 漏 3 个导入**——`CronJobInfo`/`_to_shanghai_iso` 未导入，批量创建必崩
4. **`api/settings/core.py` 漏 9 个导入**——551 passed 全绿时仍存在，靠 `check_missing_names.py` 抓到
5. **`cron/executor.py` 缺 `Any` 导入**——靠 `from __future__ import annotations` 侥幸不报错
6. **wrapper 样板冗余**（mcp）——4 个类各写一遍属性访问，已抽基类
7. **`attack` 型隐患**：`api/chat.py` 的 `_global_subagent_manager` 已集中到 `state.py`
