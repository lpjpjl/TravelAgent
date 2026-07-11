# Stage 13A：TripCoordinator happy path 与最终组装

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 13A |
| 目标 | 在无修复需求的 happy path 上串行编排 Stage 5～12B，并由 TripCoordinator 以来源 ID 回填事实、组装唯一 complete TripPlan；同步计算移出 FastAPI 事件循环 |
| 对应优化项 | P0-4 结构化输出、P0-7 深度关联化、P0-8 可执行时间轴 |
| 前置依赖 | Stage 12B |
| 后续阻断 | 未通过时不得开始 Stage 13B～13E、14、15、16、18、20、28 |
| 默认流量影响 | 无；仅显式 shadow/happy-path 验证可获得新 TripPlan。legacy/shadow 普通请求保持旧响应，new 仍 reserved |

## 2. 当前代码基线与差异

- 当前 `/trip/plan` 在 async 路由内直接运行同步 legacy MultiAgentTripPlanner，等待 MCP/LLM 时会阻塞事件循环，并由 Agent/解析 fallback 直接返回旧 `TripPlan`。
- Stage 5～12B 已逐步定义 Coordinator 所需的结构化 bootstrap、候选、两次草案、路线、时间轴、预算和校验结果，但当前没有统一入口保证 request_id、事实回填、结果状态或调用顺序。
- 当前前后端只认识 legacy schema；Stage 13A 不切默认接口和前端，先以内部对象/显式验证证明完整新链路可组装。
- 当前没有“Validator 无问题才 complete”的硬闸门，未知事实、路线缺失或校验问题可能被错误展示为成功。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 串行编排 Stage 5～12B，维护 request-scoped internal result；仅 happy path 调用 assembler | 调用顺序固定；无双跑/无后台任务/无默认 API 切换 |
| 新增 | `backend/app/services/trip_assembler.py` | 由结构化内部对象按 source ID 回填并构建 complete TripPlan | 不调用 Agent/Provider，不生成事实或修复 |
| 修改 | `backend/app/models/trip.py` | 补齐最终 TripPlan、complete status、degraded、missing items、来源提示和内部协调结果判别契约 | complete 必须全日期、valid、无 missing required item |
| 修改 | `backend/app/api/routes/trip.py` | 将新 Coordinator 同步入口放入线程池；保留 legacy/shadow/new dispatcher 语义与显式验证入口 | 事件循环不执行同步规划；new 仍返回 planner_mode_not_ready |
| 修改 | `backend/app/api/routes/__init__.py` | 注册仅本地显式 happy-path 验证入口或依赖注入装配 | 不新增普通用户默认路由 |
| 修改 | `backend/tests/integration/test_trip_coordinator_validation.py` | 补完整完整链路/组装断言 | valid 才可 complete |
| 新增 | `backend/tests/unit/services/test_trip_assembler.py` | 覆盖来源回填、状态、missing/degraded 与禁止生成字段 | 不依赖 Provider/LLM |
| 新增 | `backend/tests/integration/test_trip_coordinator_happy_path.py` | Fake Provider/LLM 下 1～3 日 complete、调用顺序、短路和无修复边界 | 默认离线、可重复 |
| 修改 | `backend/tests/api/test_trip_planner_mode.py` | 覆盖线程池、显式验证、legacy/shadow/new 与新旧响应隔离 | 普通请求无新链路双跑 |

本 Stage 不实现任何问题修复、partial/failed、前端 Pinia/UI、API 分级错误、缓存或默认切换；分别由 Stage 13B～13E、14、15、16、17 完成。legacy Agent 和旧 schema 仅作为当前模式回滚入口，不得被新链路导入。

## 4. 输入、输出与错误契约

### 4.1 Coordinator 串行调用链

对显式验证请求，Coordinator 创建 UUID4 request_id 并以同一 ID 串行调用：

```text
constraint/bootstrap → preference retrieval → primary candidates/selection
→ PrimaryDraft → lodging/meal → ItineraryDraft → RouteService/RouteSelection
→ TimelineCalculator → BudgetCalculator → TimelineValidator → TripAssembler
```

- 每个 Stage 只接收上游结构化对象及本次来源事实引用；ignored requirements、完整 Prompt、完整 payload 和 legacy 文本不进入下游；
- 任一技术错误立即传播；上游 business_infeasible/needs_repair/ValidationResult.invalid 停止后续组装，返回内部 `CoordinationOutcome.requires_repair`，不调用 TripAssembler；
- Stage 13A 不循环、替换或重建。它只处理全链路在首次计算即 valid 的情况；后续 Stage 13B～13D 在同一协调入口增量接管 invalid 结果；
- 所有外部查询仍按既有依赖同步串行。API 层使用 `anyio.to_thread.run_sync`（或等价受控线程池）运行 Coordinator，禁止在 async handler 直接执行同步 LLM/Provider 调用。

### 4.2 TripAssembler 输入与真实性

TripAssembler 仅接受同一 request_id 的 `ValidationResult.valid`、ItineraryDraft、selected RouteSegments、scheduled Timeline、BudgetCalculationResult、normalized accepted constraints 及按 ID 索引的来源 POI/酒店/天气事实：

- 每个最终景点、餐饮、酒店、路线、坐标、距离、耗时、票价与营业时间字段必须从输入来源/计算对象回填；Agent 只能贡献 Stage 8/9B 已通过校验的 reason/建议时长和 Stage 10B 选择理由；
- 任一 unknown ID、事实与决策 ID 不一致、required item missing、timeline/route 缺失、未验证状态或反向复制自由文本均为 `invalid_assembly_input`，不组装部分 TripPlan；
- 所有 unavailable 事实以 null + quality/reason 透传，不能以北京坐标、0、空字符串或 LLM 文字填充；
- `degraded=false` 固定，除非未来 Stage 17 提供经允许的 stale cache 输入；本 Stage 不自行标 degraded。

### 4.3 最终与内部结果

| 对象 | 状态 | 必填语义 |
|---|---|---|
| `TripPlan` | `plan_status=complete` | 精确覆盖请求 N 日、全部日 valid、连续 lodging、三餐、必要路线/时间轴、known budget、完整 normalized accepted constraints |
| `CoordinationOutcome.requires_repair` | 非正式内部结果 | request_id、阶段、ValidationIssue/RoutePlanningIssue、可用结构化中间结果；不含 TripPlan |
| 技术错误 | 稳定异常 | request_id/error_code/脱敏 details；不含 TripPlan |

- Stage 13A 不产生 `partial` 或 `failed` TripPlan；即使首日/后续日不可行，也只能产生 requires_repair，交 Stage 13B～13D。
- TripPlan 只含符合 Stage 2 契约的 structured data：日期、来源事实、路线/时间轴、预算假设/已知费用、状态、degraded、missing/unavailable 提示和 normalized accepted constraints；不承载完整 Prompt、原始 payload、内部 diagnostics 或 Agent 原文输出。
- Final API schema 在 Stage 16 才替换普通 `/trip/plan`。本 Stage 的显式本地验证仅返回脱敏 summary 或内部测试对象，不能让浏览器前端误消费未切换的新 schema。

### 4.4 API、模式与日志

- `legacy`：普通 `/trip/plan` 仅旧链路，Coordinator 调用数 0；`shadow`：普通请求同样仅旧链路，显式本地验证才能运行新链路；`new`：保持 503 `planner_mode_not_ready`，不得提前返回 complete。
- 显式验证入口必须为本地开发受控入口，不自动写 TripPlan/自由文本到磁盘，不用于 SSE，不增加第二次普通请求外部调用。
- 日志按 request_id 记录 stage、每阶段 status/duration、来源计数、plan_status、degraded 和稳定 error code；禁止 Key、地址、完整坐标/polyline、用户文本、Prompt、完整 payload、完整 Agent 输出和内部堆栈。

## 5. 配置与功能开关

- 不新增 Settings 或开关；沿用 `TRIP_PLANNER_MODE`、Stage 1 Provider、LLM、12A rules 配置及其已有校验。
- 显式验证入口只在本地测试/受控脚本调用，不通过环境变量向普通用户开启；默认仍 legacy。
- 线程池仅使用应用运行时受控默认限制，不增加并发查询、后台队列、任务持久化或超时配置；Stage 15/17 再统一错误/降级语义。

## 6. 实施步骤

1. 固化 complete TripPlan、CoordinationOutcome、missing/degraded 与 source ID 回填模型不变量。
2. 实现无副作用 TripAssembler：从 valid 中间对象构建唯一 complete plan，并拒绝未知/缺失/生成事实。
3. 在 Coordinator 串行接入 Stage 5～12B，固定短路点和 request_id 传播；invalid 只形成 requires_repair。
4. 在 API dispatcher 保留模式语义，将同步 Coordinator 验证调用移入线程池，新增受控本地验证装配。
5. 用 FakeAmapProvider/FakeLLM 证明 1～3 日 happy path、来源回填、无修复短路和普通流量零双跑。
6. 执行定向、Stage 1～13A 回归与前端构建，归档链路、schema、模式、线程池和回滚证据。

## 7. 明确不做

- 不替换路线、餐厅、酒店，不重排/删点，不重试修复或构造 partial/failed。
- 不把 ValidationIssue 映射 HTTP、不改前端响应类型/页面/Pinia，不切换默认接口。
- 不写数据库、历史、缓存、SSE、地图、对话或生产部署配置。
- 不恢复旧 Agent 的虚构 fallback、北京坐标或 LLM 事实补全作为任何新链路分支。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S13A-T01 | unit | valid 1/3 日中间结果 | TripAssembler complete 覆盖所有日、三餐、酒店、路线、时间轴与预算 |
| S13A-T02 | unit | 来源事实/决策 ID 回填 | POI/酒店/路线/坐标均来自输入；未知/错配拒绝 |
| S13A-T03 | unit | unavailable/known budget/degraded | null+reason 透传、known total 语义正确、degraded=false |
| S13A-T04 | unit | required/missing/timeline/route 缺失 | invalid_assembly_input，不生成部分 TripPlan |
| S13A-T05 | unit | 禁止 Agent 字段/Prompt/payload | assembler 拒绝或不序列化敏感/生成事实字段 |
| S13A-T06 | integration | 1 日 happy path | Stage 5～12B 固定顺序各一次，返回 complete 内部 plan |
| S13A-T07 | integration | 3 日/多偏好/美食餐次 happy path | 所有来源/约束/路线/时间轴完整回填 |
| S13A-T08 | integration | 任一上游 business/technical short-circuit | 后续调用数 0，无 TripAssembler/TripPlan |
| S13A-T09 | integration | Validator invalid/route needs_repair | outcome=requires_repair，无 partial/failed/complete |
| S13A-T10 | integration | request_id 传播 | 全部 fake 调用、日志摘要和 outcome 保持同一 UUID4 |
| S13A-T11 | api | legacy 普通请求 | 仅旧链路，Coordinator/线程池验证调用数 0 |
| S13A-T12 | api | shadow 普通请求/显式验证 | 普通旧响应不变；显式入口可得脱敏 happy summary |
| S13A-T13 | api | new 与线程池 | new 仍 planner_mode_not_ready；验证运行不阻塞事件循环 |
| S13A-T14 | regression | Stage 1～12B 与前端 | 上游离线测试、模式测试和生产构建不回退 |

```text
python -m pytest backend/tests/unit/services/test_trip_assembler.py backend/tests/integration/test_trip_coordinator_happy_path.py backend/tests/integration/test_trip_coordinator_validation.py backend/tests/api/test_trip_planner_mode.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、真实高德或真实 LLM。Stage 13A 必测用例数为 14。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S13A-M01 | 本地 legacy 模式用现有页面生成 Demo 行程 | 页面与旧响应可用，新 Coordinator 未执行 |
| S13A-M02 | 运行显式本地 happy-path 验证 fixture | 只显示 request_id、阶段计数、complete 结论和脱敏 summary，无敏感/完整 TripPlan 持久化 |

实际记录进入 `doc/opt_strategy/evidence/stage_13a/manual-check.md`。

## 10. 量化退出指标

- [ ] S13A-T01～S13A-T14、S13A-M01～M02 全部通过，失败数为 0，三条命令退出码为 0。
- [ ] happy path 的 Stage 5～12B 调用顺序偏差数为 0；所有 request_id 传播不一致数为 0。
- [ ] complete TripPlan 日期、酒店、三餐、必要路线、时间轴、accepted constraints 与来源 ID 回填覆盖率均为 100%。
- [ ] Validator invalid/needs_repair 被组装为 complete、partial 或 failed 的次数为 0；Stage 13A 生成 partial/failed 的次数为 0。
- [ ] legacy/shadow 普通请求 Coordinator 调用数为 0；new 提前返回新 TripPlan 的次数为 0。
- [ ] API async handler 内直接执行同步 Provider/LLM/Coordinator 的次数为 0；默认测试外部调用数为 0。
- [ ] Key、完整地址/坐标/polyline、用户文本、Prompt、payload、Agent 原文或内部堆栈进入 API/日志/证据的次数为 0；Stage 1～12B 回归和前端构建通过。

## 11. 迁移与回滚

- Stage 13A 仅增加显式新链路 complete 组装，默认 API/前端/数据均不迁移。
- 回滚先关闭显式验证装配并撤销 API 线程池调用，再移除 Coordinator happy path 与 TripAssembler；保留 Stage 5～12B 独立能力和 legacy 默认响应。
- Stage 13B 以后已依赖时，按 13D→13C→13B→13A 逆序回滚；不得通过恢复虚构 fallback 或让 invalid 直接 complete 来缩短回滚。
- 回滚后 Stage 1～12B 全量、模式/API 回归与前端构建必须通过。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_13a/test-summary.md` | 提交标识、三条命令、14 用例、离线声明和失败数 |
| 调用链矩阵 | `doc/opt_strategy/evidence/stage_13a/happy-path-chain-matrix.md` | Stage 顺序、输入/输出、短路点与测试 ID |
| 组装矩阵 | `doc/opt_strategy/evidence/stage_13a/trip-assembly-matrix.md` | TripPlan 字段、来源/计算责任、quality 与测试 ID |
| 模式/线程池记录 | `doc/opt_strategy/evidence/stage_13a/mode-threadpool-check.md` | legacy/shadow/new、显式入口、事件循环边界与测试/人工 ID |
| 人工验收 | `doc/opt_strategy/evidence/stage_13a/manual-check.md` | S13A-M01～M02 的步骤、实际结果与结论 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_13a/rollback-check.md` | 入口撤销、依赖逆序、命令和结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 12B 实际 Done，并在开工前复核 Stage 5～12B 的模型/错误/模式契约、TripPlan schema、线程池装配以及 Stage 13B～13E 的接管边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
