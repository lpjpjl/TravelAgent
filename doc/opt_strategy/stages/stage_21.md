# Stage 21：本地开发前端代理统一

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 21 |
| 目标 | 在新增图片批量、PDF 上传/下载和 SSE 前，统一前端业务请求入口与 Vite 本地开发代理，消除绕过代理的硬编码后端 host |
| 对应优化项 | P2-6 前端代理统一，并为 Stage 23、25～28 的图片、报告、流式请求提供请求边界 |
| 前置依赖 | Stage 19；开工前复核 `frontend/vite.config.ts`、`frontend/src/services/api.ts`、Result 图片请求、地图 Key 配置和后端 `/api` 路由前缀 |
| 后续阻断 | 未通过时不得开始 Stage 23 图片批量接口、Stage 25 PDF 导出上传接入、Stage 27 历史入口或 Stage 28 SSE 前端请求 |
| 默认流量影响 | 只改变本地开发请求入口和前端调用路径；不改变后端 API schema、TripPlan、地图 Key、安全来源或生产部署方案 |

## 2. 当前代码基线与差异

- 当前 `frontend/vite.config.ts` 已有 `/api -> http://localhost:8000` 的本地代理，但 `frontend/src/services/api.ts` 仍通过 `VITE_API_BASE_URL || http://localhost:8000` 形成浏览器侧后端 host 依赖。
- 当前 `frontend/src/views/Result.vue` 的图片加载直接 `fetch("http://localhost:8000/api/poi/photo?...")`，绕过 Vite 代理和统一错误处理。
- 当前 `services/api.ts` 主要服务 JSON 请求，尚未为 Stage 28 的 `text/event-stream`、Stage 26/27 的 PDF multipart 上传/下载和 Stage 23 的图片批量请求保留清晰边界。
- Stage 19 已规划前端地图 Key 风险治理；Stage 21 不改变 `VITE_AMAP_WEB_JS_KEY`，也不把地图 Web JS 加载纳入后端代理。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `frontend/vite.config.ts` | 统一本地开发 `/api` 代理目标、超时和必要的代理日志/错误边界 | 本地 dev 下业务 API 均走同一 `/api` 前缀 |
| 修改 | `frontend/src/services/api.ts` | 移除浏览器侧硬编码后端 host；建立 JSON 请求客户端和稳定错误解码入口 | `generateTripPlan` 等 JSON API 不再依赖 `VITE_API_BASE_URL` |
| 新增/修改 | `frontend/src/services/request.ts` 或等价模块 | 预留 `jsonRequest`、`streamRequest`、`multipartRequest`、`downloadRequest` 的职责边界 | Stage 23/25/27/28 可接入，不提前实现业务功能 |
| 修改 | `frontend/src/views/Result.vue` | 图片请求改走统一 API 边界或 `/api` 相对路径 | 不再出现 `http://localhost:8000/api/poi/photo` |
| 修改 | `frontend/.env.example` | 删除或标注 `VITE_API_BASE_URL` 不作为本地业务 API 默认项；保留 `VITE_AMAP_WEB_JS_KEY` 语义 | 不新增生产 base URL 设计 |
| 新增 | `doc/opt_strategy/evidence/stage_21/` | 保存扫描、构建、本地代理冒烟和回滚证据 | 证据不含真实 Key、token 或完整请求体 |

若当前项目没有前端单元测试框架，本 Stage 采用 `npm --prefix frontend run build`、源码扫描和记录化人工验收；不为代理统一单独引入新测试框架。

## 4. 请求与代理契约

### 4.1 本地开发代理

| 项 | 约束 |
|---|---|
| 前端业务 API 前缀 | 浏览器代码统一使用相对路径 `/api/...` |
| Vite dev 代理 | 仅在本地开发中将 `/api` 转发到后端 `http://localhost:8000` 或本地配置的等价目标 |
| 生产部署 | 本 Stage 不设计生产 `base URL`、反向代理、CDN 或跨域策略 |
| CORS | 不扩大后端 CORS 白名单；本地代理应减少浏览器直接跨域依赖 |
| 非业务外部资源 | 高德 Web JS、地图瓦片、图片 CDN URL 等第三方公开资源不经后端 `/api` 代理，除非后续 Stage 明确设计 |

- 业务请求禁止在 Vue 组件、Store 或服务模块中拼接 `http://localhost:8000`、`http://127.0.0.1:8000`、公网 host 或端口。
- 允许 `frontend/vite.config.ts` 和本地说明中出现代理目标 host；该 host 不进入浏览器运行时代码或构建产物。
- `VITE_API_BASE_URL` 不作为本地开发业务请求默认依赖。若未来需要生产 base URL，必须由独立 Stage/部署设计显式引入。

### 4.2 请求客户端边界

| 请求类型 | 当前 Stage 责任 | 后续消费 |
|---|---|---|
| JSON | 正式落地统一客户端、超时、错误 envelope 解码和脱敏日志 | Stage 16/15 响应语义、Stage 23 图片批量 JSON |
| Stream | 只预留独立函数/类型和禁止 axios 拦截器误处理的边界；不实现 `/trip/plan/stream` | Stage 28 |
| Multipart upload | 只预留独立函数/类型和 header 规则；不实现 PDF 上传 API | Stage 26/27 |
| Download blob | 只预留独立函数/类型和错误处理边界；不实现历史下载 UI | Stage 26/27 |

- JSON 客户端不得把 `Content-Type: application/json` 强加给 multipart、blob 下载或 stream。
- Stream 请求后续必须使用 `fetch + ReadableStream`，不得被 axios JSON client 包装为普通 JSON。
- 请求日志只记录 method、相对路径、稳定 code、request_id 和耗时，不记录完整 TripPlan、自由文本、PDF Blob、图片关键词、后端堆栈或 Key。

### 4.3 路由覆盖范围

Stage 21 统一这些业务请求路径的前端入口命名和代理策略：

| 领域 | 路径示例 | 本 Stage 行为 |
|---|---|---|
| trip | `/api/trip/plan` | 迁移到统一 JSON client |
| map/poi | `/api/map/*`、`/api/poi/*` | 迁移组件内硬编码调用，图片单张接口先维持现有语义 |
| image batch | `/api/poi/photos/batch` 或后续确定路径 | 仅预留命名与 client 类型，不新增后端接口 |
| stream | `/api/trip/plan/stream` | 仅预留 stream client 边界，不新增后端接口 |
| report | `/api/reports/*` 或后续确定路径 | 仅预留 multipart/download client 边界，不新增后端接口 |

## 5. 配置与功能开关

不新增功能开关。

| 配置 | 所属 | 本 Stage 处理 |
|---|---|---|
| `VITE_AMAP_WEB_JS_KEY` | 前端公开地图 Key | 沿用 Stage 19，不经 `/api` 代理，不改名、不复用后端 Key |
| `VITE_API_BASE_URL` | 旧/可疑前端 API base | 本地业务请求不得依赖；如文件中存在，删除或标注废弃，不作为默认路径 |
| Vite proxy target | 本地开发配置 | 仅存在于 `vite.config.ts` 或本地开发说明，不进入浏览器业务代码 |

## 6. 实施步骤

1. 梳理全部前端业务请求调用点，区分业务 API、地图 Web JS、第三方图片 URL 和静态资源。
2. 建立或整理统一请求模块：JSON 客户端正式可用，stream/multipart/download 只保留边界和类型约束。
3. 将 `generateTripPlan`、图片获取等现有业务请求迁移到相对 `/api` 路径，删除组件内硬编码后端 host。
4. 收敛 Vite 本地代理配置和 `.env.example`，明确本 Stage 不处理生产 base URL。
5. 执行源码/构建产物扫描、前端 build、本地 dev 代理冒烟和回滚记录。

## 7. 明确不做

- 不新增或修改后端图片批量、SSE、PDF 上传/下载、历史报告或生产部署接口。
- 不改变 `/api/trip/plan` 的请求/响应 schema、TripPlan、错误 envelope、Store 语义或规划链路。
- 不改变地图 Key 治理、Web JS 加载、安全来源白名单、路线 polyline 或地图组件行为。
- 不引入并行请求、后台任务、EventSource、服务端推送或 PDF Blob 生成。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S21-T01 | static | 前端源码业务请求 host 扫描 | 除 `vite.config.ts`/文档外，无 `localhost:8000`、`127.0.0.1:8000` 硬编码业务调用 |
| S21-T02 | build | 前端生产构建 | 构建通过，bundle 中无本地后端 host |
| S21-T03 | unit/static | JSON client | `generateTripPlan` 使用相对 `/api/trip/plan`，错误 envelope 不泄漏原始对象 |
| S21-T04 | static | multipart/stream/download 边界 | JSON 默认 header 不污染 stream、multipart 或 blob 下载函数 |
| S21-T05 | regression | 地图 Key | `VITE_AMAP_WEB_JS_KEY` 仍只用于地图 Web JS，不被请求 client 或后端代理替代 |
| S21-T06 | regression | 后续路径预留 | Stage 23/26/27/28 可在统一 client 下扩展，无需再次引入全局 base URL |

```text
npm --prefix frontend run build
rg -n "http://localhost:8000|http://127\\.0\\.0\\.1:8000|VITE_API_BASE_URL" frontend/src frontend/dist
rg -n "fetch\\(|axios\\.create|baseURL|FormData|ReadableStream|text/event-stream|blob" frontend/src
```

扫描命令在实施时需结合最终代码人工判读：`vite.config.ts` 中的本地 proxy target 可以存在；业务运行时代码和构建产物不得依赖硬编码后端 host。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S21-M01 | 启动后端和 Vite dev，提交一次现有行程规划 | 浏览器请求为 `/api/trip/plan`，由 Vite 代理转发，功能不回退 |
| S21-M02 | 打开结果页并触发当前单张图片加载 | 图片请求走 `/api/poi/photo` 或统一 API 边界，无硬编码 host |
| S21-M03 | 临时停止后端再提交请求 | 前端显示稳定错误，不暴露后端 host、堆栈或完整 payload |
| S21-M04 | 检查地图加载 | 地图 Web JS Key 路径保持 Stage 19 语义，不经 `/api` 代理 |

## 10. 量化退出指标

- [ ] S21-T01～T06、S21-M01～M04 通过，命令退出码为 0。
- [ ] 前端业务运行时代码和构建产物中硬编码本地后端 host 命中数为 0。
- [ ] 组件内直接业务 `fetch` 绕过统一边界的新增数量为 0；既有图片请求已迁移。
- [ ] 因 JSON client header 污染 stream/multipart/blob 的路径数为 0。
- [ ] 因本 Stage 修改导致 TripPlan schema、地图 Key 语义或后端 API 行为变化的次数为 0。

## 11. 迁移与回滚

- 迁移顺序：先建立统一请求模块和代理契约，再迁移 `generateTripPlan`，最后迁移 Result 图片请求和清理 `.env.example`。
- 回滚时先恢复调用方，再恢复请求模块；不得回滚为组件内新增更多硬编码 host。
- 回滚后必须重新运行 S21-T01～T03、前端 build 和 M01，确认现有行程生成仍可用。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_21/test-summary.md` | build、扫描命令、T01～T06 结果和失败数 |
| 代理冒烟 | `doc/opt_strategy/evidence/stage_21/proxy-smoke.md` | dev 代理目标、浏览器相对路径、后端响应结论 |
| 请求边界矩阵 | `doc/opt_strategy/evidence/stage_21/request-client-matrix.md` | JSON/stream/multipart/download 职责、后续 Stage 映射 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_21/manual-check.md`、`rollback-check.md` | M01～M04、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 21 独立一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 19 实际 Done，并在开工前复核 Vite 代理、前端请求调用点、地图 Key 配置和后续 Stage 23/25～28 的接口边界。当前不表示 Ready、Done、Gate D、设计冻结或允许修改工程代码。
