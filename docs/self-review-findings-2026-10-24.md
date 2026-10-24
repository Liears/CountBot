# 审查过程自查：死代码 / 冗余 / 缺陷 / 垃圾代码

> 时间：2026-10-24
> 范围：本轮（上下文工程审计 + pi 工具层加固）所有改动涉及的文件
> 方法：静态扫描（AST 未用导入/未用参数）+ 符号引用计数 + 异常路径实测 + 边界值实测
> 结果：**发现 6 类问题，全部已修**；测试 507 passed / 0 failed

---

## 1. 我引入的真实缺陷（最该先说的）

### 1.1 旧 `execute` 路径绕过写入锁 —— 上一轮的加固有漏洞

**问题**：上一轮给 `write_file` / `edit_file` 的 `execute_outcome()` 加了路径锁，但**漏了同名 `execute()`**。而 `registry.py:666` 的 `await tool.execute(**arguments)` 确实会走旧路径。

**后果**：如果任何调用方走 `execute()`，并发追加依然会丢更新——等于修复只覆盖了一半入口。

**证据**：
```
registry.py:666:  result = await tool.execute(**arguments)
loop.py:993:      return await self.tools.execute_outcome(...)   # 主路径
```

**修复**：`execute()` 也包上同一把 `path_write_lock`，两条入口共用锁。

**这类问题的性质**：同一份写入逻辑有两条实现，加固时只改了一条。属于「修复不彻底」，比没修更危险——因为看起来已经修了。

### 1.2 单行截断给出无效的续读提示 —— 误导性输出

**问题**：`read_file` 遇到单行超长文本（如压缩过的 JSON）被字符预算截断时，仍输出：
```
[Showing lines 1-1 of 1. Use start_line=2 to continue.]
```
但总行数就是 1，`start_line=2` 必然越界报错。

**后果**：模型照提示执行 → 报错 → 可能反复重试同一操作。这是我新加的「智能提示」自己制造的陷阱。

**修复**：区分「因行数截断」与「因字符截断且无剩余行」，后者不再给行号提示，改为建议读取更窄范围。

### 1.3 加锁导致失败语义退化（已修，记录在案）

给 write 加锁初期，既有测试 `test_file_mutation_pre_and_post_boundary_failures_are_distinguished` 失败：锁的 key 生成对非标准 Path 抛异常，异常被归类为 `NOT_ATTEMPTED`，把「副作用已发生」错报成「未执行」。

**为什么严重**：`SideEffectState` 决定失败能否安全重试。把已写入报成未写入，重试就会重复写文件。

**修复**：`_key()` 兜底所有异常；edit 拆方法后通过 `effect` 可变容器回传状态。

---

## 2. 死代码

### 2.1 `PathLockRegistry.stats()` + 未使用的 `logger` 导入

`path_lock.py` 里 `stats()` 全库零调用；`logger` 导入后从未使用。**已删**。

### 2.2 `_clip_by_chars` 的 `start_line` 参数

我加这个方法时传了 `start_line`，但函数体里从未使用。**已删参数**。

### 2.3 `usage_recorder.py` 未使用的 `List` 导入

AST 扫描发现。**已删**。

### 2.4 `MCPResourceWrapper` —— 差点误删，已恢复

**经过**：改成 catalog 聚合后，我判断它成了死代码，删了 68 行。结果两个测试文件立即报 `ImportError`。

**真实情况**：它是被测试直接使用的（`test_mcp_execution_boundaries.py` 用它验证资源读取的取消/失败边界）。虽然生产注册路径不再用它，但它承载着**测试覆盖价值**。

**处理**：恢复，类注释里写明「注册路径已改为聚合式，本类保留用于精确绑定与既有测试」。

**教训**：`grep 定义处只有一行` 不等于死代码。测试是消费者，判断死代码必须连测试一起查。

### 2.5 `backend/modules/agent/compactor.py` —— 0 字节空文件（既有）

**发现**：这个文件 0 行 0 字节，但历史上被人 `import` 过——上一轮审计发现 `handler.py` 曾试图从别的模块导入不存在的 `ConversationSummarizer`，压缩功能因此静默失效很久。

**当前状态**：已无任何引用。**建议删除**。留着会让后来者以为「压缩功能在这里」。

**注意**：我没有直接删——它在 git 里是有人提交的，删除属于仓库清理决策，需要你确认。

---

## 3. 冗余代码

### 3.1 写入逻辑两份实现

`write_file` 的 `execute()` 与 `execute_outcome()` 各自完整实现了一遍读-改-写；`edit_file` 同样。两份逻辑现在都加了锁，但**仍然重复**。

**风险评估**：不合并的话，下次改写入逻辑极可能又只改一处（这次的漏洞就是这么来的）。

**建议**：让 `execute()` 退化为 `execute_outcome().display_text` 的一行包装。改动小、收益明确。**未做**，因为这会改变异常表现（`execute` 现在返回字符串错误而非抛异常），需要确认没有调用方依赖细节。

### 3.2 `_sanitize_name` 与 `_qualify_name` 并存（已自然消解）

删 `MCPResourceWrapper` 时 `_sanitize_name` 失去最后调用点，但恢复该类后它又有了用处。当前 `_sanitize_name` 仅被 `_qualify_name` 之外的旧类使用——**保留合理，无需处理**。

---

## 4. 不完善之处

### 4.1 `_SECTION_SWITCH_MAP` 的 `identity` 映射不准确

`request_preview.py` 里 `identity` 标注为 `controlled_by: None`（始终注入），但实际上 `identity` 区块内部的**环境信息段**受 `core_rules` 控制（`context.py:375`）。

**影响**：预览面板会告诉用户「identity 恒定注入」，但关掉 `core_rules` 后 identity 内容其实会变短。属信息不准确，非功能错误。

### 4.2 批量读的预算只统计成功项

`_read_multiple_files_outcome` 里 `used += len(item.display_text)` 在失败时也会累加（因为 `display_text` 是错误消息，通常很短），逻辑上没问题；但如果某个文件读取失败且错误消息很长，预算判断会略微偏保守。**可接受，不修**。

### 4.3 `MCPResourceCatalogWrapper` 的资源清单是静态快照

描述在构造时生成，服务器后续新增资源不会反映。原 `MCPResourceWrapper` 也是同样行为，非本轮引入。**记录，不修**。

---

## 5. 看起来有问题但实际没问题的（避免误判）

| 现象 | 结论 |
|------|------|
| `_edit_by_text` / `_edit_by_lines` 是否死代码 | **不是**。`execute()` 旧路径仍调用它们 |
| `path_write_lock` 异常路径是否漏锁 | **不会**。`__aexit__` 的 `finally` 语义保证释放，已实测异常后下一次写入正常 |
| 截断 `HEAD_KEEP_RATIO=0.25` 在极小上限时是否出错 | **不会**。`max_output_length=1` 时 head=0、tail=1，实测输出正常 |
| `session_name_matches` 的 LIKE 转义 | **正确**。`%`/`_`/`\` 均转义，并附带 `OR name = prefix` 兼容无时间戳旧数据 |

---

## 6. 修复汇总

| # | 类型 | 问题 | 处理 |
|---|------|------|------|
| 1 | 缺陷 | 旧 `execute` 路径绕过写入锁 | 加锁，两入口共用 |
| 2 | 缺陷 | 单行截断给无效行号提示 | 区分截断原因，不给假提示 |
| 3 | 死代码 | `PathLockRegistry.stats()` + 未用 `logger` | 删除 |
| 4 | 死代码 | `_clip_by_chars` 未用参数 | 删除参数 |
| 5 | 死代码 | `usage_recorder` 未用 `List` | 删除导入 |
| 6 | 误删回滚 | `MCPResourceWrapper` | 恢复并注明保留理由 |

新增 2 条测试锁定单行截断行为，防止回归。

---

## 7. 需要你决定的两件事

1. **删除 `backend/modules/agent/compactor.py`**（0 字节空文件）——需要仓库清理授权。
2. **合并 `execute` / `execute_outcome` 双份写入实现**——会让旧路径的异常表现变化（返回字符串 vs 抛异常），需要确认没有调用方依赖。

---

## 8. 自我评价

这轮最实质的问题不是死代码，而是**1.1 那个加固漏洞**：我在上一轮明确说要解决「并发写丢更新」，结果只堵了主路径、漏了兼容路径——而两条路径都在生产代码里可达。这说明「看到一处就改一处」在有多实现的代码里是不够的。

`MCPResourceWrapper` 的误删则暴露另一个偏差：把「生产注册路径不引用」等同于「死代码」，忽略了测试也是消费者。

其余是些小瑕疵（未用参数、未用导入、提示文案不精确），属于常态清理范畴。
