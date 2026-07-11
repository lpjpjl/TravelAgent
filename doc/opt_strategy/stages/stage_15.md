# Stage 15：分级错误处理

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 15 |
| 目标 | 将技术错误、受控业务不可行和额外要求 `parse_failed` 明确分流为稳定 API/前端语义，不再将所有异常包装为 500 或将技术故障伪装为 failed |
| 对应优化项 | P1-2 分级错误处理、P1-1/Stage 16 的状态切换前置 |
| 前置依赖 | Stage 13D、13E |
| 后续阻断 | 未通过时不得开始 Stage 16、17、28；Stage 20 的请求错误消费也必须复用本契约 |
| 默认流量影响 | 可统一现有 API 错误 envelope，但不切换新成功 schema、不使 `new` 模式可用；legacy/shadow 普通成功流量保持原入口 |

## 2. 当前代码基线与差异

- 当前 `trip.py` 捕获所有异常、打印完整 traceback 并以 500/字符串 detail 返回；`ErrorResponse` 没有 request_id、retryable 或分级详情，前端 axios 直接暴露后端 detail。
- 13D 已能在可信输入下形成 complete/partial/failed，但技术 Provider/LLM/schema/config 错误仍应传播；13E 只有本地 generic error，14 只负责展示结果状态。
- Stage 17 才实现真实缓存/stale。本 Stage 只用 fake cache source 验证“可信 stale 可继续、无可信输入则技术错误”的边界，不新增缓存、TTL 或降级策略。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/trip.py` | 固化 `ApiErrorResponse(request_id,error_code,message,retryable,details?)` 与可公开 detail 白名单 | 无 stack/key/free text/payload，JSON 稳定 |
| 新增 | `backend/app/errors.py` | 定义 `ConfigurationError`、`UpstreamTimeoutError`、`ProviderUnavailableError`、`UpstreamRateLimitError`、`CoreOutputError` 等技术分类 | 异常保留 cause 仅供内部日志 |
| 新增 | `backend/app/api/error_mapping.py` | 单一 HTTP 映射与 request_id 提取/生成 | API 路由不再手写 500 分支 |
| 修改 | `backend/app/api/main.py`、`api/routes/trip.py` | 注册异常处理、在线程池/Coordinator 边界保持 request_id | legacy/new 预备入口均返回相同错误 envelope |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 只将可信业务不可行交给 13D 状态机，技术错误保持类型化传播 | 禁止技术错误构造 failed plan |
| 修改 | `frontend/src/types/index.ts`、`services/api.ts` | 解码稳定 error envelope，丢弃未白名单字段 | 不向 Store 传 Axios/后端原始对象 |
| 修改 | `frontend/src/stores/current_trip.ts`、`views/Home.vue` | 映射 retryable/config/later 业务展示文案；保留原 plan | 不自动重试、不暴露堆栈 |
| 新增 | `frontend/src/components/TripRequestErrorNotice.vue` | 独立渲染技术错误的稳定文案与返回表单入口 | 不依赖 Stage 14 的结果状态组件 |
| 新增 | `backend/tests/api/test_trip_error_mapping.py`、`backend/tests/integration/test_coordinator_error_split.py` | 覆盖 HTTP/status/request_id 与技术/业务分流 | fake Provider/LLM/cache，离线 |
| 新增 | `doc/opt_strategy/evidence/stage_15/` | 记录 API、前端人工和回滚证据 | 脱敏 |

## 4. 输入、输出与错误契约

### 4.1 唯一分流表

| 类别 | 例子 | HTTP/响应 | 前端行为 |
|---|---|---|---|
| 输入错误 | 请求字段、枚举、日期不合法 | 422 `validation_error` | 标注表单，不提交 plan |
| 配置错误 | provider/key/规则配置缺失或非法 | 503 `configuration_error`，`retryable=false` | 提示检查本地配置，不清空旧 plan |
| 超时 | Provider/LLM 在有限重试后超时且无可信输入 | 504 `upstream_timeout`，`true` | 稍后重试入口 |
| 限流/Provider 不可用 | 当前 Provider 限流、网络/服务不可用 | 503 `upstream_rate_limited`/`provider_unavailable`，`true` | 稍后重试入口 |
| 核心输出非法 | PrimaryDraft/ItineraryDraft/RouteSelection 解析重试耗尽、未知 ID | 502 `core_output_invalid`，`true` | 通用上游数据异常提示 |
| 模式未就绪 | Stage 16 前 `new` | 503 `planner_mode_not_ready`，`false` | 仅开发提示，不回退旧/新混跑 |
| 额外要求解析耗尽 | `ExtraConstraint` 无法确认 | 不是错误；`parse_failed` 进入 ignored 后继续 | 结果中按既有说明，不显示技术失败 |
| 业务不可行 | 合法空候选、无必要路线、有限修复后不可行 | HTTP 200，`TripPlan.plan_status=partial/failed` | 保留结果状态供 Stage 14/16 消费，不映射为技术错误 |
| 可信 stale | fake/live 失败但有满足契约的 stale 输入 | 不是错误；plan `degraded=true` | 显示来源时效提示 |

- 技术错误绝不返回 `TripPlan`，业务 `failed` 绝不携带技术 stack；两种语义必须由响应 shape 和 status code 区分。
- 成功/业务结果沿用 Stage 13D/16 的 `TripPlan`；本 Stage 不引入第二个 success envelope。所有错误统一为 `{request_id,error_code,message,retryable,details?}`，`details` 只允许字段名、稳定子码和安全 remediation，不含地址、坐标、自由文本、Key、token、payload、Prompt、provider 原文或 exception repr。
- API 层生成或接受 UUID4 `request_id` 并在线程池、Coordinator、Provider、Agent、日志和错误 envelope 传播；异常处理器不得生成与请求不同的 ID。

### 4.2 Coordinator、缓存与前端边界

- Coordinator 只把已有 `BusinessInfeasible`/Validator 结果交 Stage 13D 产生 complete/partial/failed；不得用 catch-all 将 `ConfigurationError`、Provider/LLM/schema 错误改写为 failed。
- fake stale 只能证明输入携带来源、过期时间和 `freshness=stale` 时能够继续形成 `degraded=true`；fake 无 stale、坏 stale 或错误 schema 必须走技术错误。真实缓存键、容量、TTL 与读取优先级完全留给 Stage 17。
- `api.ts` 只把白名单 error 转为 `CurrentTripError`；Store 用 sequence 调 `failRequest`，保留旧完整 plan。retry button 仅回到/聚焦原表单，不能自动重放自由文本或后台重试。
- 13E 的 `storage_unavailable` 仍只是本地持久化记录，不应变为后端 API 错误；`TripRequestErrorNotice` 不显示业务 plan status，Stage 14 的 status banner 若已存在也不显示技术错误详情。

## 5. 配置与功能开关

不新增 Settings、环境变量或功能开关。沿用 `TRIP_PLANNER_MODE`：`legacy/shadow` 继续默认旧成功响应，`new` 在 Stage 16 前固定 `planner_mode_not_ready`；错误 envelope 的统一不是默认新链路切换。

## 6. 实施步骤

1. 固定公开错误模型、稳定 code、HTTP 映射和 request_id 传播测试。
2. 在 Provider/Agent/Coordinator 边界建立类型化技术异常；移除路由 catch-all traceback/detail 外泄。
3. 收口业务不可行与 fake stale 的输入/输出边界，确保它们不进入技术错误或虚构 fallback。
4. 前端集中解码 envelope 并映射可展示文案/重试提示，保持 Store 原子性。
5. 执行 API、集成、全量后端与前端 build，完成 UI/回滚记录。

## 7. 明确不做

- 不实现缓存、stale 持久化、请求重试策略、SSE、统一 API client 重构或默认新链路切换。
- 不更改 complete/partial/failed 业务判定、重排/repair、正式 TripPlan schema 或展示布局。
- 不向用户开放 traceback、Prompt、自由文本、完整上游响应或内部异常类型。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S15-T01 | api | 422 请求校验 | `validation_error`、request_id、无 plan |
| S15-T02 | api | 配置/Provider/限流/超时 | 分别 503/503/503/504，retryable 正确 |
| S15-T03 | api | 核心 Agent/schema 非法 | 502 `core_output_invalid`，无内部 details |
| S15-T04 | integration | 合法空/repair 不可行 | 200 partial 或 failed，不被映射 API error |
| S15-T05 | integration | 技术错误 | 原样分类，绝不构造 failed TripPlan |
| S15-T06 | integration | fake stale/no stale | 前者 degraded plan，后者技术错误 |
| S15-T07 | api | request_id/敏感信息 | UUID 一致，无 key/prompt/payload/stack |
| S15-T08 | regression | legacy/shadow/new | 普通成功语义不变，new 仍 not_ready |

```text
python -m pytest backend/tests/api/test_trip_error_mapping.py backend/tests/integration/test_coordinator_error_split.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S15-M01 | 模拟 timeout/rate-limit/configuration error | 显示对应可展示提示/重试性，不泄露栈且旧行程保留 |
| S15-M02 | 模拟 business partial/failed 与 fake stale | 前者保留为结果状态、后者只携带 degraded，均不进入技术错误组件 |
| S15-M03 | 检查浏览器 Network 与页面 | error envelope 有 request_id/code；页面无 payload/Prompt/异常正文 |

## 10. 量化退出指标

- [ ] S15-T01～T08、S15-M01～M03 通过，三条命令退出码为 0。
- [ ] 未分类 500、技术错误携带 TripPlan、业务不可行携带 HTTP 技术错误的次数均为 0。
- [ ] request_id 不一致、敏感信息出现在 envelope/日志/前端的次数为 0。
- [ ] fake stale 被标 error 或无 stale 被伪装 degraded 的次数为 0；普通请求双跑次数为 0。

## 11. 迁移与回滚

- 迁移先上线错误模型/handler 与前端解码，再让路由使用分类错误；不改变 success response 和默认 mode。
- 回滚先回滚 Stage 16，再移除 handler/前端映射；临时恢复 legacy 错误时仍不得恢复完整 traceback 或虚构 fallback。
- 回滚后重跑 S15-T01～T05、build 和 M01，记录 `rollback-check.md`。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_15/test-summary.md` | 命令、8 用例、离线、失败数 |
| 错误矩阵 | `doc/opt_strategy/evidence/stage_15/error-matrix.md` | 类别、HTTP、code、retryable、UI 与测试 |
| 人工验收 | `doc/opt_strategy/evidence/stage_15/manual-check.md` | M01～M03 脱敏结果 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_15/rollback-check.md` | 逆序、命令、无敏感确认 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批 Stage 14～16 交接审计；执行 Ready 待 R1**。R1 必须等待 Stage 13D、13E 实际 Done，并复核 error model、request_id、业务/技术分流、fake stale 边界及 Stage 16 的 response 切换。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
