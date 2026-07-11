# Stage 23：图片批量接口

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 23 |
| 目标 | 将结果页 N 个景点图片 HTTP 请求收敛为一次或少数几次批量请求，并保证单项失败不拖垮整批 |
| 对应优化项 | P2-4 图片批量接口 |
| 前置依赖 | Stage 13A、21；开工前复核前端请求统一入口、当前 `/api/poi/photo`、UnsplashService、缓存策略和正式 TripPlan 点位 ID/名称 |
| 后续阻断 | 未通过时不得开始 Stage 24 图片加载优化，也不得开始依赖图片 ready 的 Stage 25 PDF 导出 |
| 默认流量影响 | 只改变图片查询请求形态；不改变 TripPlan、点位排序、图片 CDN 资源、导出逻辑或规划链路 |

## 2. 当前代码基线与差异

- 当前 `Result.vue` 遍历全部景点并为每个景点发起单独 `GET /api/poi/photo?name=...`，浏览器请求数随景点数线性增长。
- 当前前端以景点名称作为 `attractionPhotos` key，重名景点、跨天重复和 TripPlan 替换时容易串扰。
- 当前后端 `get_attraction_photo(name)` 同步调用 Unsplash，失败时抛 500；没有批量上限、单项错误、去重、结果映射或部分成功语义。
- Stage 21 已规划统一请求边界；Stage 23 必须通过该边界接入，不重新引入硬编码 host。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/api/routes/poi.py` | 新增图片批量接口，保留单张接口兼容 | 批量请求返回 item 级结果 |
| 修改 | `backend/app/services/unsplash_service.py` | 暴露批量处理辅助，复用单项搜索和缓存 | 单项失败不抛掉整批 |
| 新增/修改 | `backend/app/services/image_cache.py` 或等价模块 | 进程内图片查询缓存、去重和 TTL | 重复点位不重复外部查询 |
| 修改 | `frontend/src/services/api.ts` 或 Stage 21 请求模块 | 新增 `fetchAttractionPhotosBatch` | 走 `/api` 相对路径和统一错误边界 |
| 修改 | `frontend/src/views/Result.vue` 或图片 composable | 一次行程一次或分块请求图片映射 | 不再按景点逐个 HTTP 请求 |
| 修改 | `frontend/src/types/index.ts` | 定义图片批量请求/响应类型 | 与后端 contract 对齐 |
| 新增 | `backend/tests/api/test_poi_photos_batch.py` | 覆盖上限、去重、单项失败、部分成功 | fake Unsplash，离线 |
| 新增 | `doc/opt_strategy/evidence/stage_23/` | 保存接口契约、测试、请求数和回滚证据 | 脱敏，不含真实图片 API Key |

## 4. 图片批量接口契约

### 4.1 请求

建议接口：

```text
POST /api/poi/photos/batch
```

请求体：

| 字段 | 类型 | 约束 |
|---|---|---|
| `items` | array | 1～`PHOTO_BATCH_MAX_ITEMS` |
| `items[].client_ref` | string | 前端生成的稳定引用，通常为 day key + attraction id/name |
| `items[].poi_id` | string? | 有正式 POI ID 时优先使用 |
| `items[].name` | string | 无 POI ID 时用于查询，长度有限 |
| `items[].city` | string? | 可选，提高搜索相关性 |
| `items[].category` | string? | 可选，不得作为事实推断 |

- 前端必须先按 `poi_id || normalized(city/name/category)` 去重，再发送批量请求。
- 后端必须再次去重，防止前端 bug 导致重复外部查询。
- 请求不得包含完整 TripPlan、自由文本偏好、用户备注或 Key。

### 4.2 响应

响应体：

| 字段 | 类型 | 语义 |
|---|---|---|
| `success` | bool | HTTP 层成功收到批量处理结果；不代表每项都有图片 |
| `request_id` | string? | 若 Stage 15 已提供则透传 |
| `data.items` | array | 与输入 `client_ref` 可映射的结果 |
| `data.summary` | object | total/succeeded/failed/cached |

单项结果：

| 字段 | 类型 | 语义 |
|---|---|---|
| `client_ref` | string | 前端映射 key |
| `status` | `success`/`not_found`/`failed`/`skipped` | 单项状态 |
| `photo_url` | string? | 可公开图片 URL |
| `source` | string? | 如 `unsplash`、`cache` |
| `error_code` | string? | 脱敏稳定错误码 |

- 单项 `not_found` 或 `failed` 不让整个批次返回 500。
- 只有请求体非法、超出批量上限、认证/配置级不可用且无可返回项等情况，才按 Stage 15 错误 envelope 返回请求级错误。
- 返回顺序可不同于输入，前端必须按 `client_ref` 映射。

### 4.3 上限、超时与缓存

| 项 | 约束 |
|---|---|
| 单批上限 | 默认 30，作为配置常量或服务内常量登记；超过返回 400 |
| 单项超时 | 不高于现有 Unsplash 单项超时，失败写入单项 `failed` |
| 总批次耗时 | 有上限；不得无限串行等待 |
| 缓存 | 复用 Stage 17 思路的进程内有界 TTL 缓存或本 Stage 专用轻量缓存 |
| 并发 | 默认仍同步串行处理；不引入后台任务或跨请求并发队列 |

## 5. 配置与功能开关

不新增用户功能开关。

| 配置/常量 | 默认 | 说明 |
|---|---|---|
| `PHOTO_BATCH_MAX_ITEMS` | 30 | 单请求最大图片项数 |
| `PHOTO_CACHE_TTL_SECONDS` | 待 Stage 17 ADR/本 Stage 证据确定 | 图片 URL 查询缓存 TTL |
| `PHOTO_ITEM_TIMEOUT_SECONDS` | 10 或沿用现有服务 | 单项外部查询超时 |

若新增环境变量，必须登记到 04 配置表并在 `.env.example` 中只放占位和说明，不放真实 Key。

## 6. 实施步骤

1. 定义图片批量请求/响应 Pydantic 和前端 TypeScript 类型。
2. 后端新增批量接口：校验上限、规范化 key、去重、查询、单项错误捕获和 summary。
3. 接入有界 TTL 缓存，避免重复景点/重复刷新造成重复外部查询。
4. 前端从 TripPlan 派生图片请求项，按一次行程一次或分块调用批量接口。
5. 保留单张接口兼容，但结果页不再默认使用 N 个单张请求。
6. 完成离线 fake 测试、前端 build、Network 请求数人工验收和回滚证据。

## 7. 明确不做

- 不实现图片懒加载、骨架屏、重试状态机、`imagesReady` 或导出等待；这些属于 Stage 24。
- 不改变图片来源服务商、不申请 Key、不提交真实图片 API Key。
- 不把图片 URL 写入 TripPlan 事实字段或后端持久化存储。
- 不改变点位筛选、排序、地图、时间轴、预算或 PDF 导出。
- 不引入异步后台任务、并行 Agent 或浏览器端无限并发预加载。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S23-T01 | api | 3 个不同点位批量请求 | 返回 3 个 item，summary 正确 |
| S23-T02 | api | 重复点位 | 后端只调用一次 fake provider，多个 client_ref 均有映射 |
| S23-T03 | api | 单项 not_found/failed | 批次 HTTP 成功，失败项有稳定 status/error_code |
| S23-T04 | api | 超过批量上限 | 返回 400 或统一 validation error |
| S23-T05 | api/cache | 缓存命中 | 第二次请求不访问 fake provider，source/freshness 可追踪 |
| S23-T06 | frontend/static | Result 图片请求 | 不再对每个景点调用 `/api/poi/photo` |
| S23-T07 | build | 前端构建 | build 通过，无硬编码 host |

```text
python -m pytest backend/tests/api/test_poi_photos_batch.py -q
npm --prefix frontend run build
rg -n "/api/poi/photo|photos/batch|fetchAttractionPhotos" frontend/src backend/app
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S23-M01 | 打开典型 2～3 天行程并观察 Network | 图片 API 请求数显著收敛为 1 次或少数分块请求 |
| S23-M02 | 构造重复景点名或同一 POI 跨天出现 | 前后端均去重，页面映射不串图 |
| S23-M03 | fake/拦截某一项失败 | 其他图片正常显示，失败项进入 Stage 24 可消费状态 |
| S23-M04 | 后端图片 Key 缺失或 provider 不可用 | 不泄露 Key 或堆栈，页面不崩溃 |

## 10. 量化退出指标

- [ ] S23-T01～T07、S23-M01～M04 通过，命令退出码为 0。
- [ ] 典型报告浏览器图片业务 API 请求数较逐项请求显著下降，目标为每个 TripPlan 1 次，超过上限时按分块次数。
- [ ] 单项失败导致整批失败的次数为 0。
- [ ] 重复点位重复外部查询次数为 0。
- [ ] 真实 Key、完整用户输入、完整 TripPlan 出现在日志/证据中的次数为 0。

## 11. 迁移与回滚

- 迁移顺序：后端 contract → fake 测试 → 缓存/批量处理 → 前端 service → Result 接入。
- 回滚时先让前端回到单张接口，再禁用批量接口调用；保留单张接口兼容，避免结果页图片完全不可用。
- 回滚后运行 S23-T01～T04、前端 build 和 M01，确认请求形态与页面降级可控。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_23/test-summary.md` | 后端测试、前端 build、扫描和 T01～T07 结果 |
| 接口契约 | `doc/opt_strategy/evidence/stage_23/photos-batch-contract.md` | 请求/响应字段、上限、单项状态 |
| 请求数记录 | `doc/opt_strategy/evidence/stage_23/network-request-count.md` | 旧/新请求数、分块说明 |
| 缓存/失败矩阵 | `doc/opt_strategy/evidence/stage_23/cache-error-matrix.md` | cache hit、not_found、failed、provider unavailable |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_23/manual-check.md`、`rollback-check.md` | M01～M04、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 22～24 跨 Stage 一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 13A 和 Stage 21 实际 Done，并在开工前复核请求统一入口、图片 Provider、缓存边界和正式 TripPlan 点位标识。当前不表示 Ready、Done、Gate D、设计冻结或允许修改工程代码。
