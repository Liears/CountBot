# 拆分文件 API 实测记录（第二轮）

> 时间：2026-10-24 16:56–17:05
> 范围：上一轮未覆盖的拆分包端点——cron 全链路、settings 剩余、chat 剩余
> 结果：**全部通过，未发现新缺陷**；重点验证了当年拆分时出过问题的两条路径

---

## 1. cron API（`api/cron/` 拆分，8 文件）

| 端点 | 结果 |
|------|------|
| `GET /jobs` | ✓ 列表正常 |
| `POST /jobs` | ✓ 创建成功，`next_run` 带 +08:00 时区 |
| `GET /jobs/{id}` | ✓ |
| `PUT /jobs/{id}` | ✓ 改 schedule 生效 |
| `POST /jobs/batch` | ✓ **success_count: 2** |
| `POST /jobs/batch-delete` | ✓ deleted_count: 2 |
| `POST /jobs/{id}/run` | ✓ 提交执行 |
| （执行后）`GET /jobs/{id}` | ✓ `run_count: 1, last_status: ok, last_error: None` |
| `GET /jobs/{id}/messages` | ✓ 响应结构正常 |
| `POST /validate` | ✓（上轮已测） |
| `DELETE /jobs/{id}` | ✓ 清理 |

**重点**：`batch` 两条路径就是当年 `check_missing_names.py` 抓到
漏导入 `CronJobInfo` / `_to_shanghai_iso` 的地方——当时 508 个测试全绿也
没发现，因为批量创建路径无测试覆盖。这次在真实 HTTP 上完整跑通，
且 `_to_shanghai_iso` 产出的 `next_run` 时区正确。

**手动执行是一次真实 LLM 调用**（executor.py 拆分端到端跑通）。

---

## 2. settings API（`api/settings/` 拆分，10 文件）

| 端点 | 结果 |
|------|------|
| `GET /providers` | ✓ 22 个 provider |
| `GET /security/dangerous-patterns` | ✓ |
| `GET /external-coding-tools` | ✓ 返回 profiles 配置 |
| `POST /external-coding-tools/check` | ✓ 假命令 → `{"success":false,"message":"未找到可执行命令"}` |
| `GET /workspace/info` | ✓ 98 个文件 |
| `POST /workspace/clean-temp` | ✓ 无 24h 以上临时文件 |
| `GET /export` | ✓ version 1.0.0，12 个 config 键 |
| `POST /import`（原样导回） | ✓ 200 |
| `POST /test-connection`（假 key） | ✓ 结构化失败：`{"success":false,"error":"API 密钥无效..."}` |

**未测两处（有意跳过）**：
- `POST /workspace/select-directory`——会弹原生文件对话框，干扰桌面
- `POST /workspace/set-path`——触发工作区热重载，破坏性

---

## 3. chat API（`api/chat/` 拆分，13 文件）剩余端点

| 端点 | 结果 |
|------|------|
| `PUT /sessions/{id}/summary` | ✓ 字段 `summary` 有 `min_length=10`：6 字被 422 拒（校验生效），15 字 200 |
| `DELETE /sessions/{id}/summary` | ✓ |
| `GET/PUT/DELETE /sessions/{id}/config` | ✓ 会话级配置读写与重置全通 |
| `POST /sessions/{id}/attachments` | ✓ 见下方契约说明 |
| `GET .../attachments/{path}` | ✓ 下载内容与上传**逐字一致** |
| `GET /sessions/{sid}/messages/{mid}/tool-calls` | ✓ 真实数据：message 40 有 6 条工具调用，分页信封完整 |

**附件端点的契约**（测试时踩到，记录下来）：不是 multipart，而是
**原始字节流 + `x-file-name` 请求头**（头部值需 URL-encode，否则中文文件名
乱码——我第一次没编码，存储名变成 `æµ_è.txt`，属客户端未按契约，非服务缺陷）。
按正确契约重传后 `content_type: text/plain` 正确、下载内容逐字一致。

---

## 4. 排查过程中的两个"假警报"

1. **tool-calls 404**：先以为是缺陷，实际是我上一轮清理测试会话时把
   含消息 47/48 的会话级联删除了。换用历史真实数据后通过。
2. **`messages` 表无 `tool_calls` 列**：工具调用记录在 `tool_conversations`
   表，`message_id` 关联。用 `tool_conversations.message_id` 找到真实数据后通过。

---

## 5. 累计覆盖情况

两轮 API 实测覆盖了四个拆分包的几乎全部 HTTP 端点：

- `api/cron/`（12 端点）：**12/12**
- `api/settings/`（14 端点）：12/14（select-directory、set-path 有意跳过）
- `api/chat/`（23 端点）：21/23（workspace 文件下载需工作区文件、
  sessions_updated WebSocket 事件属推送通道）
- `modules/mcp/`：无 HTTP 端点（纯内部模块），由单元测试覆盖

上一轮发现并修复的「先落库后校验」缺陷，本轮回归确认修复有效。
单元测试 651 passed，测试数据已全部清理（会话、cron 任务均删除）。
