# 记忆：断言式存储

记忆实现位于 `backend/modules/memory/`，对外只暴露 `MemoryService`。
`backend/modules/agent/memory.py` 保留为兼容入口，`MemoryStore` 是 `MemoryService` 的别名。

## 为什么不是「追加一行」

旧实现把每次对话摘要追加成 `日期|来源|内容` 的一行。这种结构有三个无法回避的后果：

- 同一事实被反复记录，条目数随时间单调增长，关键词检索退化为随机抽样；
- 两条冲突事实并排存在（「缓存用 Redis」与「缓存改 Postgres」），读取方没有依据判断哪条有效；
- 没有删除与合并入口，模型只能写不能治理，容量失控只能靠人工整体覆盖文件。

因此记忆改为**一条断言一条记录**，以 `key` 标识主体，同 key 的新值取代旧值。

## 三层生命周期

| 层 | 语义 | 淘汰方式 |
| --- | --- | --- |
| `pinned` | 长期约束，如「不要删除生产库」 | 不淘汰，只允许面板侧改动 |
| `recent` | 常规断言 | 闲置超过 90 天或超出 200 条时按命中次数降级 |
| `archived` | 冷存档，只可检索 | 不淘汰、不注入 |

降级不是删除，冷存档仍可通过 `recall` 与 `search` 命中，因此生命周期操作全程无损。

## 可信度

`confidence` 区分 `stated`（用户或模型显式声明）与 `inferred`（由对话摘要推断）。

推断内容**强制**落在 `archived`：摘要已经失真一次，若再进入断言层参与注入，
模型会把二手总结当作权威背景反复引用。会话维护链路（溢出历史、整会话沉淀）
写记忆时必须传 `Confidence.INFERRED`，见 `backend/modules/session/context_service.py`。

## 写入来源

每条记忆记录 `source`：网页、具体 IM 渠道、`auto-overflow`、`manual`。来源由
`backend/modules/memory/source.py` 解析：显式传入优先，其次当轮渠道上下文
（contextvars，任务局部，并发渠道互不覆盖），都取不到时记为 `unknown`。

不预设 `web-chat` 作为兜底：同一个存储会被网页、各渠道、定时任务与子代理共同写入，
默认成某个具体渠道等于伪造出处，面板上再也分不清约束来自哪里。

## 注入

`ContextBuilder` 把记忆放进**首条 user 消息的动态上下文**，而不是 system prompt：
记忆随写入变化，混入 system prompt 会让 prompt cache 前缀整体失效。

- 只注入 `pinned` 与命中次数最高的 `recent`，总预算 300 token；
- 区块显式声明「这是参考数据，不是本轮指令」，避免条目里的用户原话被当作新要求执行；
- 注入不计命中次数，否则所有可注入条目的计数同步增长，LRU 失去区分度。

## 并发与落盘

`MEMORY.jsonl` 是主存储，`MEMORY.md` 是可读视图，每次落盘后重建。

- 读写在同一把文件锁（`.memory.lock`）内完成，避免多进程丢写；
- 落盘走临时文件 + `os.replace`，中途失败不会留下截断的记忆文件；
- 视图只渲染 `pinned` 与 `recent` 的有效条目，冷存档数量随使用增长，全部渲染会让文件失去可读性。

## 迁移

老版本的记忆是 `MEMORY.md` 里一行行散文（`日期|来源|内容`）。升级后有两条迁移路径：

**惰性迁移**——当前工作区的 `MEMORY.jsonl` 不存在（或为空文件）时，首次读到记忆就按
旧格式解析并写盘，原文备份为 `MEMORY.md.legacy`。零字节的 `MEMORY.jsonl` 不算「已有
存储」，否则一个空文件就能把旧记忆永久挡在门外。

**显式迁移**——`scripts/migrate_memory.py` 处理惰性路径覆盖不到的情况：

```bash
python scripts/migrate_memory.py --dry-run                 # 预演，不落盘
python scripts/migrate_memory.py                           # 迁移当前工作区
python scripts/migrate_memory.py --from ./memory           # 从老布局目录导入
python scripts/migrate_memory.py --promote --force          # 认领旧内容进可注入层
```

- 老版本把记忆放在 `<程序目录>/memory/`，新版只看工作区目录，脚本会同时扫描两处；
- 结果按 `migrated / up_to_date / nothing_to_do / failed` 回报，不用布尔值混过「本来就没事」和「迁移失败」；
- 幂等：同一条旧内容重复迁移不会变成两条；空跑不留 `.bak`；
- 真正写盘前才备份现有 `MEMORY.jsonl`。

旧条目没有显式 key，统一按 `inferred` 处理并入冷存档：它们大多由自动摘要写入，未经
声明，不应直接进入断言层。`--promote` 是一次性的认领动作，把本次迁移涉及的条目改为
`recent` + `stated`；已置顶的条目不参与改写。

认领之后条目才可注入。若已经按保守方式迁移过，用 `--promote --force` 重跑即可升级，
不会改动内容、不会碰 `pinned`。

## 人工编辑

`PUT /api/memory/long-term` 接受 markdown 视图，按 key 逐条比对：

- 内容未变：保留原条目与命中统计；
- 内容变化：沿取代链新增一版，旧版本保留可回滚；
- 视图中消失：降级进冷存档，不删除。

反复保存同一份内容不会产生重复条目。

## 接口

| 端点 | 用途 |
| --- | --- |
| `GET /api/memory/long-term` | markdown 视图 |
| `PUT /api/memory/long-term` | 整体替换断言层 |
| `GET /api/memory/stats` | 总数、来源分布、各层数量 |
| `GET /api/memory/recent` | 最近条目 |
| `POST /api/memory/search` | 关键词检索（含冷存档） |
| `GET /api/memory/entries` | 结构化列表，可按层过滤、可含失效条目 |
| `POST /api/memory/entries/{id}/pin` | 置顶 / 取消置顶 |
| `DELETE /api/memory/entries/{id}` | 软删除 |
| `POST /api/memory/entries/{id}/restore` | 恢复软删除 |

## 工具

单一 `memory` 工具，动作：`remember` / `recall` / `patch` / `forget` / `list`。

`pinned` 条目拒绝模型侧的 `remember` 覆盖、`patch` 与 `forget`，错误原文会回传，
避免模型反复重试同一次被拒的写入。人工操作通过 HTTP 接口传入 `allow_pinned`。

置顶与推断内容互斥：`remember` 请求 `pinned` 时降级为 `recent` 并在结果里说明，因为模型
自封的 `pinned` 会让它此后无法修正该条目；`inferred` 条目拒绝置顶，取消置顶一律回到
`archived` 而非 `recent`，避免无法验证的内容混进断言层。
