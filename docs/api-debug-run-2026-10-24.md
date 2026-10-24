# 实际 API 调测记录

> 时间：2026-10-24 16:20–16:40
> 方式：启动真实服务，curl 直打 HTTP 端点；含一次真实 LLM 调用
> 结果：**发现并修复 1 个真实缺陷（含数据污染事故），其余全部通过**

---

## 0. 发现的缺陷：设置更新「先落库、后校验」

**现象**：`PUT /api/settings` 传 `{"security":{"audit_log_enabled":"maybe"}}`
返回 500，错误信息还是误导性的 `Failed to get settings`。

**根因链**（比表面严重）：

1. `UpdateSettingsRequest` 的各分区是裸 `dict`，Pydantic 在入口不校验
2. `update_settings` 把脏值写进 config 后**先落库**（`save_config`，第 492 行）
3. 最后构造 `SettingsResponse` 时才校验失败 → 被兜底 except 包成 500
4. **此时脏值已持久化**——我复测时把 `audit_log_enabled='maybe'` 写进了 DB
5. 下次启动 `AppConfig(**config_dict)` 解析失败 → **服务起不来**

即：一个 400 级别的输入错误，最终把整个服务弄挂了。

**修复**（`api/settings/core.py`）：

- 落库前加 `AppConfig.model_validate(config.model_dump())`，脏值在保存前拦截
- `get_settings` / `update_settings` 的 except 链在通用 `Exception` 之前
  先捕获 `ValidationError` → 400，错误信息可直接读

**验证**：PUT 脏值 → 400；GET 仍 200；DB 值不变（原子性成立）。

**数据修复**：污染的 `config.security.audit_log_enabled` 已从 `'maybe'`
恢复为 `'true'`（修复前有快照，DB 备份在 `/tmp/countbot_backup_*.db`），
并全库排查了同类布尔键，无其他脏值。

---

## 1. 通过的端点

| 端点 | 结果 |
|------|------|
| `GET /api/health` | ✓ |
| `GET /api/settings` | ✓ 分区完整 |
| `PUT /api/settings` 非法布尔 | ✓ 400「配置项无效: Input should be a valid boolean」 |
| `PUT /api/settings` 非法正则 | ✓ 400「custom_deny_patterns[0] 无效: ...」 |
| `PUT /api/settings` 合法值 | ✓ 200 |
| `POST /api/cron/validate` 坏表达式 | ✓ 200 + `valid:false` |
| `POST /api/cron/validate` 好表达式 | ✓ 200 + `valid:true` + 下次运行时间（带 +08:00 时区） |
| `POST /api/chat/sessions` | ✓（注意：`name` 是 query 参数） |
| `POST /api/chat/preview` | ✓ 含 token 分桶 |
| `POST /api/chat/send` | ✓ SSE 全链路：`start` → 流式 `message` → `done`，消息落库（message_id 47/48） |
| `GET /api/chat/sessions/{id}/messages` | ✓ 用户与助手消息完整 |
| `DELETE /api/chat/sessions/{id}` | ✓ 200 |
| `GET /api/system/usage` | ✓ 78 次请求 / 148,606 tokens |
| `GET /api/system/concurrency` | ✓ 闸门统计全零（单请求无排队，符合预期） |

**preview 的 token 分桶实测**：

```
est_total_tokens: 3812
  system_prompt        1693 tok  44.4%
  system_tool_schemas   416 tok  10.9%
  messages             1703 tok  44.7%
```

**SSE 实测**（真实 LLM，zhipu/deepseek-v4.1-flash）：

```
event: start   data: {"message_id": "47"}
event: message data: {"content": "2"}
event: message data: {"content": "，绷不住了..."}
event: done    data: {"message_id": "48"}
```

---

## 2. 说明与备注

- **`cache_read_tokens: 0`**：当前 zhipu 上游对此模型未上报缓存明细字段，
  解析逻辑无米下锅，0 是正确行为而非缺陷。切换到上报 `cached_tokens` 的
  上游/模型后即有数据。
- **API 契约与直觉不同的两处**（测试时踩到，非缺陷）：
  `POST /api/chat/sessions` 的 `name` 是 query 参数；
  `POST /api/chat/preview` 的 `session_id` 必填。
- **环境注意**：本机 shell 配了 `HTTP_PROXY=127.0.0.1:64621`，curl 打
  localhost 必须加 `--noproxy '*'`，否则被代理拦截返回 502。
- 并行会话正在重构 channels（handler 拆出 commands.py 等），本次启动已
  加载其最新代码，启动与应用层无异常。

---

## 3. 回归

单元测试 651 passed；服务日志在修复后的运行期间无新增 ERROR。
