# Stage 10A：RouteService 固定交通基础版

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 10A |
| 目标 | 基于完整 `ItineraryDraft` 的真实坐标，由 RouteService 查询每个必要相邻路段的真实候选、TripCoordinator 确定性选择固定交通 `RouteSegment`；路线缺失必须显式交给后续修复，禁止直线或生成式兜底 |
| 对应优化项 | P0-6 RouteService 基础版、P0-7 行程深度关联化，并为 P0-8/P1-4 提供真实路线输入 |
| 前置依赖 | Stage 4、Stage 9B（Stage 5 的 accepted 移动约束经 Stage 9B 上游链传递） |
| 后续阻断 | 未通过时不得开始 Stage 10B、11、12A、12B、13A～13D、16、18、20、28 |
| 默认流量影响 | 无；只在显式 Coordinator 新链路验证中生成内部路线结果。legacy/shadow 普通请求仍不运行 Coordinator，`new` 仍为 reserved |

## 2. 当前代码基线与差异

- 当前 `backend/app/services/amap_service.py::plan_route()` 以地址字符串调用 MCP，成功和失败分别固定返回 `{}`；它不接受 Stage 2 的 `GeoPoint`、不输出可用路线事实，也没有选择或失败语义。
- Stage 4 完成后，新的 `AmapService.plan_routes()` 将可按坐标和 `walking/driving/transit` 查询返回 `list[SourcedRoute]`，但不解释用户偏好、步行上限、公交步行接驳，也不决定最终路段。
- 当前不存在 `RouteService`、固定方式回退、相邻活动枚举、路线选择策略、公交预计出发时刻或“必要路线缺失”的结构化输入；旧前端仅以景点坐标直线连线。
- Stage 9B 的目标 `ItineraryDraft` 已固定每日起止酒店、三餐和全部活动顺序，但尚未把 ID 回填为来源坐标，也未计算任何路线。`lodging_side` 早餐不属于外出活动，不能形成路线端点。
- Stage 11 才生成真实时间轴和实际出发时刻；本 Stage 只生成公交首次查询的可复现预计时刻，并提供一次刷新接口，不实施时间轴或触发刷新。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|------|------|---------------|----------|
| 新增 | `backend/app/services/route_service.py` | 展开必要相邻端点、同步查询固定方式候选并提供公交刷新接口 | 不导入 Agent、旧 `schemas.py` 或前端代码；候选均引用 `SourcedRoute.route_id` |
| 新增 | `backend/app/services/route_policy.py` | 提供供 Coordinator 调用的固定偏好过滤与选择纯函数：方式映射、walking/公交接驳过滤和稳定排序 | 输入候选相同时结果完全一致；无配置读取或外部调用 |
| 修改 | `backend/app/models/route.py` | 补齐 `RoutePlanningResult`、`RoutePlanningIssue`、`RouteQueryReference` 和刷新计数的严格契约 | 能表达 ready、需要修复、首次/刷新公交查询；不改写 Stage 4 的来源路线事实 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 9B 成功后调用 RouteService，并以 RoutePolicy 完成固定回退选择、保存内部路线结果或修复输入 | 普通 legacy/shadow 请求调用数为 0；不组装 `TripPlan` |
| 新增 | `backend/tests/unit/services/test_route_policy.py` | 覆盖查询顺序、方式映射、边界过滤和稳定排序 | 所有固定偏好、并列候选和约束边界可离线判定 |
| 新增 | `backend/tests/unit/services/test_route_service.py` | 使用 FakeAmapProvider 验证端点展开、查询、结果、刷新和错误传播 | 默认公网请求数为 0，禁止直线兜底 |
| 修改 | `backend/tests/unit/models/test_route_timeline.py` | 覆盖新路线规划结果、问题与刷新计数模型序列化 | 非法状态组合和负刷新计数被拒绝 |
| 新增 | `backend/tests/integration/test_trip_coordinator_routes.py` | 覆盖 Stage 9B→10A 接入、上游短路和模式回归 | Coordinator 为下游 Stage 11 保存唯一内部结果 |

本 Stage 不修改 `backend/app/services/amap_service.py`、Provider/解析器、DestinationResearchAgent、ItineraryPlannerAgent、Stage 9A 住宿餐饮选择、TimelineCalculator、BudgetCalculator、API 路由、legacy Agent、前端类型或页面。Stage 10B 才实现 `mixed` 的三方式候选集和受限 Agent 选择。

## 4. 输入、输出与错误契约

### 4.1 入口与输入快照

TripCoordinator 只在以下前置全部成立时调用 `RouteService.query_fixed_route_candidates()`，并在本进程内将候选交给 `RoutePolicy` 选择：

- Stage 9B 成功输出完整 `ItineraryDraft`；上游 `business_infeasible` 或技术错误由 Coordinator 原样短路，RouteService/AmapService 调用次数为 0；
- 提供请求的 `request_id`、城市/adcode（供 transit Provider 使用）、`TransportationPreference`、已 accepted 的 `mobility_constraint`，以及本次草案全部活动、酒店与来源 POI 的 ID→`GeoPoint` 只读映射；
- 所有必须端点均有真实 GCJ-02 坐标。未知 ID、重复 endpoint reference、缺酒店坐标、POI 坐标 unavailable 或 `lodging_side` 被作为外出活动均为 `invalid_itinerary_route_endpoint` 技术校验错误；不得调用地理编码、LLM、默认城市坐标或坐标平移修补；
- 本 Stage 只接受 `walking`、`public_transit`、`self_drive`。`mixed` 必须由 Coordinator 路由至尚未就绪的 Stage 10B，当前返回稳定 `route_mode_not_ready`，不得伪装成任一固定方式。

输入快照仅含稳定 ID、日期、活动顺序/类型、早餐类型、来源坐标、交通偏好、有效步行上限和 transit 所需城市标识；不得写入自由文本、完整候选、完整 Provider payload、Prompt 或 Key。

### 4.2 必要端点与路段枚举

RouteService 对每个 `ItineraryDraftDay` 先将 `lodging_side` 早餐从外出序列排除，再按草案既有顺序建立外出活动序列。每条 `RouteRequirement` 都有稳定的 `day_index`、`leg_index`、`origin_ref`、`destination_ref` 与 `required=true`：

1. `start_hotel_id → first_external_activity_id`；如早餐为 `gourmet`，它就是首个外出活动；如为 `lodging_side`，首段直达首个景点/午餐/晚餐活动。
2. 同日相邻两个外出活动之间各一段，严格保持 Stage 9B 顺序，不因距离、类别或偏好重排。
3. 非末日 `last_external_activity_id → end_hotel_id`；`end_hotel_id` 必须等于次日 `start_hotel_id`。末日不得生成返店段，且末日最后外出活动必须是晚餐，错误留给 Stage 9B/11 的既有校验处理。

每个返回日必须至少有一个外出活动；不满足时为 `invalid_itinerary_route_endpoint`。`lodging_side` 早餐既不产生 `RouteRequirement`、`RouteSegment`、公交查询，也不以“酒店到酒店”补写虚构路段。

### 4.3 预计公交出发时刻

- 只有 Provider 查询方式为 `transit` 时，RouteService 才传递 `RouteQueryReference.departure_at`；时区固定 `Asia/Shanghai`，日期取所属 `ItineraryDraftDay.date`。
- 初始时钟：`gourmet` 早餐首段为 08:00；`lodging_side` 早餐后的首段为 09:00。随后每一段的预计出发时刻等于前一段已选择路线的预计到达时刻，加前一外出活动的 `suggested_duration_minutes`，再加 30 分钟独立缓冲。它仅按 day part、餐次槽位和既有活动顺序推导查询参考，不是最终时间轴、不校验餐窗，也不得进入最终 API 展示。
- 每段保存首次查询的 `query_departure_at`、查询日期和 `refresh_count=0`。Stage 11 若计算出的同一 transit 路段实际出发与此值绝对差超过 30 分钟，可调用 `refresh_transit_segment()` 一次；该方法只重查该段 transit、保持相同端点和有效约束，成功后 `refresh_count=1`。计数已为 1 时返回 `transit_refresh_limit_reached`，不再请求。
- 本 Stage 不创建 TimelineCalculator、不比较实际时刻、不自动刷新或循环重排；Stage 11 接入时必须证明“至多一次重查后重新计算时间轴”的调用次数上限。

### 4.4 固定方式查询、过滤与选择

有效步行上限 `effective_walking_limit_meters = min(2000, accepted mobility_constraint.max_walking_segment_meters)`；没有 accepted 约束时为 2000。边界为闭区间：距离 `<=` 上限合格，`>` 上限淘汰。

| 请求偏好 | 首次查询/实际方式 | 仅在无合格首选时按此顺序补查/复用 | 不允许 |
|----------|-------------------|-----------------------------------------|--------|
| `walking` | `walking → walking` | `driving → taxi`，`transit → transit` | `self_drive` |
| `public_transit` | `transit → transit` | `walking → walking`，`driving → taxi` | `self_drive` |
| `self_drive` | `driving → self_drive` | `walking → walking`，同一 driving 事实可复用为 `taxi`，再查 `transit → transit` | 将 taxi 标成 self_drive，或假定无选择自驾用户拥有其他车辆 |

- RouteService 每次 `AmapService.plan_routes()` 调用保持同步串行。对于同一端点、Provider 查询方式、策略和 transit 查询参考的重复需要，必须复用本次调用结果，禁止重复外部查询；不引入跨请求缓存、并发、重试或 Provider 自动切换。
- TripCoordinator 按表中顺序请求 RouteService，并对每批 `SourcedRoute` 调用无副作用的 RoutePolicy。RoutePolicy 先按 `(duration_seconds, distance_meters, route_id)` 选定同一 Provider 查询方式的最佳候选，再按相同三元组从合格方式中确定最终候选；`route_id` 是唯一最后决胜键。
- RoutePolicy 仅在 `walking` 候选真实 `distance_meters <= effective_walking_limit_meters` 时视其合格。步行超限只淘汰该候选，Coordinator 仍须继续查询/选择定义的 taxi 或 transit；不得直接判定该日失败。
- RoutePolicy 仅在 transit 候选具备来源提供的每一个步行接驳段距离、且每段均 `<= effective_walking_limit_meters` 时视其合格。缺少接驳距离、存在负值或任一段超限时淘汰该 transit 候选，并记录稳定过滤原因；不得以总距离、名称猜测或 LLM 推断代替接驳事实。
- driving 事实映射为 `taxi` 或 `self_drive` 仅改变业务实际方式，不改变来源 route ID、polyline、距离、耗时、来源策略或真实性等级。固定方式不计算打车价格、自驾费用、换乘惩罚或多目标评分。

### 4.5 输出、失败与日志

TripCoordinator 在 RoutePolicy 选择后生成 `RoutePlanningResult`；它只是一项内部协调结果，不是 `TripPlan`：

| 状态 | 输出 | 语义 |
|------|------|------|
| `ready` | 按 `day_index/leg_index` 排序的完整 `RouteSegment` 集合；每条含端点引用、实际方式、距离、耗时、polyline、route ID、来源/策略和 `RouteQueryReference` | 每个 `RouteRequirement` 恰有一条真实、合格路线 |
| `needs_repair` | 不返回部分路线集合；返回一个按 `day_index/leg_index` 排序的 `RoutePlanningIssue[]` | 至少一条必要路段在所有允许候选都合法空、被硬约束过滤或无合格来源路线；稳定 `error_code=required_route_unavailable` |

- `needs_repair` 不等同于最终 `PlanStatus.partial/failed`。Stage 13B/13D 才按修复顺序处理路线、餐厅、酒店和日期；本 Stage 不删除活动、不替换事实、不回退直线、也不把部分成功路段暴露给下游。
- ProviderError、AmapParseError、输入模型错误和 `transit_refresh_limit_reached` 均是技术/校验错误，原样传播；合法空路线或合格候选耗尽才可形成 `required_route_unavailable`。两类不得互相转换。
- 日志只记录 request_id、stage=`routes_fixed`、day/leg、请求偏好、查询方式、候选/淘汰计数、最终实际方式、route ID、refresh_count、耗时与稳定错误码。不得记录 Key、地址、完整坐标、完整 polyline、完整 payload、自由文本或完整草案。

## 5. 配置与功能开关

- 本 Stage 不新增 Settings、环境变量或用户流量开关。`MAX_WALKING_SEGMENT_METERS=2000` 是 Stage 2 已冻结领域常量；accepted 移动约束只能进一步收紧，不能放宽。
- 沿用 `AMAP_PROVIDER`、`AMAP_API_KEY` 及 Stage 1 Provider capability；RouteService、RoutePolicy 和模型均不直接读取 Settings。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 的普通请求不运行本 Stage；显式 shadow 验证可调用到内部结果；new 在 Stage 16 前仍返回 `planner_mode_not_ready`。
- 路线查询次数、公交刷新上限和稳定排序是受测试锁定的领域规则：固定方式每段按 4.4 串行查询，单段 transit 刷新最多 1 次。不提供环境变量覆盖。

## 6. 实施步骤

1. 在 `models/route.py` 固化路线需求、查询参考、成功结果、需要修复问题与刷新计数的封闭模型；补齐 JSON 正反例和状态互斥校验。
2. 在 `route_policy.py` 固化供 Coordinator 使用的三种固定偏好查询顺序、Provider 查询方式→实际方式映射、2000m/accepted 上限、公交接驳字段校验及稳定排序。
3. 在 `route_service.py` 实现只读端点解析与逐日路段展开，严格排除 `lodging_side` 早餐，验证酒店/活动坐标及非末日/末日边界。
4. 实现同步串行的候选查询和请求内去重；RouteService 只返回 Stage 4 的真实 `SourcedRoute` 候选，RoutePolicy 只过滤/选择并映射为 `RouteSegment`，不调用 Agent、不生成路线或直线。
5. 实现 `RouteQueryReference` 的初始时钟、transit 查询参数及仅一次的 `refresh_transit_segment()`；不接入时间轴刷新循环。
6. 在 TripCoordinator 的 Stage 9B 成功路径接入 RouteService 和 RoutePolicy：按固定方式顺序查询并确定性选择，ready 时保存内部结果，needs_repair 时保存结构化修复输入；不改变普通 API、legacy/shadow/new dispatcher 或最终计划组装。
7. 用 FakeAmapProvider 与固定 ItineraryDraft fixture 完成单元、模型和集成测试；执行 Stage 1～10A 回归及前端构建，归档路线、约束、调用次数和回滚证据。

## 7. 明确不做

- 不实现 `mixed` 路线多候选查询、RouteSelection Agent、Agent JSON mode 或无效 Agent 选择回退；属于 Stage 10B。
- 不生成最终时刻、餐窗、wait、buffer、冲突或公交自动刷新循环；属于 Stage 11/12B。
- 不计算票价、打车费、自驾费用、油费、停车费或预算；属于 Stage 12A。
- 不替换路线之外的餐厅/酒店、重排/删点、生成 partial/failed，或组装/返回 `TripPlan`；属于 Stage 13A～13D/16。
- 不缓存、stale 降级、并发、Provider 自动切换、地图显示、前端直线替换、SSE 或 API schema 变更；分别属于 Stage 17、20、28 等后续 Stage。
- 不调用地址地理编码、使用默认北京坐标、估算直线距离、补写 polyline 或将 Provider 技术错误伪装为可修复的无路线。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S10A-T01 | unit | 1～3 日、住宿侧/美食级早餐的路段展开 | 必要端点顺序正确；lodging_side 不产生路段；非末日返店、末日无返店 |
| S10A-T02 | unit | 未知 ID、缺坐标、重复端点和非法早餐类型 | `invalid_itinerary_route_endpoint`，Amap 调用数为 0 |
| S10A-T03 | unit | walking 偏好、1999m/2000m/2001m | 前两者可选；2001m 淘汰并按 taxi→transit 查询 |
| S10A-T04 | unit | public_transit 首选成功/无合格候选 | 先 transit；仅失败后按 walking→taxi 回退，从不选择 self_drive |
| S10A-T05 | unit | self_drive 首选成功/回退 | driving 映射 self_drive；失败后按 walking→taxi→transit，driving 事实可复用为 taxi |
| S10A-T06 | unit | accepted 1200m 移动约束 | walking 1200m 合格、1201m 淘汰；约束不会放宽系统上限 |
| S10A-T07 | unit | transit 步行接驳 | 每段均在上限内才合格；缺距离、负值或任一超限均淘汰 |
| S10A-T08 | unit | 单方式多条来源路线与跨方式并列 | 均按 `(duration_seconds, distance_meters, route_id)` 稳定选择 |
| S10A-T09 | unit | 同段重复查询需要 | 同一请求内按端点/方式/策略/查询时刻复用，外部调用数精确为 1 |
| S10A-T10 | unit | 固定方式全部合法空/全部被过滤 | `needs_repair + required_route_unavailable`；不返回部分段、直线或默认方式 |
| S10A-T11 | unit | ProviderError/AmapParseError | 错误原样传播，不转换 `needs_repair`，不继续伪造候选 |
| S10A-T12 | unit | mixed | `route_mode_not_ready`，不查询 walking/driving/transit |
| S10A-T13 | unit | transit 首次预计时刻 | gourmet 08:00、lodging_side 09:00；后续时刻由已选段、建议时长和 30 分钟缓冲确定 |
| S10A-T14 | unit | transit 刷新 | 差异触发入口一次；刷新后计数为 1；第二次为 `transit_refresh_limit_reached` |
| S10A-T15 | contract | ready 路线结果 | 所有 requirement 恰有一条来源 route ID、GCJ-02 polyline、方式/策略和查询参考 |
| S10A-T16 | contract | needs_repair 路线结果 | 只含稳定排序 issue；禁止 partial RouteSegment、最终时刻、费用或 TripPlan 字段 |
| S10A-T17 | unit | 模型序列化 | 负距离/耗时、负刷新计数、ready 与 issue 并存、needs_repair 含 segments 均拒绝 |
| S10A-T18 | integration | Stage 9B→10A happy path | Coordinator 保存完整内部结果，供 Stage 11 消费 |
| S10A-T19 | integration | Stage 9B 上游不可行/技术错误 | RouteService/Amap 调用数为 0，状态原样短路 |
| S10A-T20 | integration | needs_repair 接入 | Coordinator 保存修复输入，不生成 TripPlan、partial 或 legacy fallback |
| S10A-T21 | regression | legacy/shadow/new 模式 | legacy/shadow 普通请求 RouteService 调用数为 0；new 仍 `planner_mode_not_ready` |
| S10A-T22 | regression | Stage 1～9B 与前端 | 上游离线测试和生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/models/test_route_timeline.py backend/tests/unit/services/test_route_policy.py backend/tests/unit/services/test_route_service.py backend/tests/integration/test_trip_coordinator_routes.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、真实高德或真实 LLM。Stage 10A 必测用例数为 22，失败数必须为 0。

## 9. 人工验收

不适用：本 Stage 不接入普通 API/UI，不应消耗真实路线配额。固定偏好、接驳上限、回退、刷新、错误与调用次数均由 FakeAmapProvider、固定草案和离线 fixture 自动化覆盖；Stage 11/20 才分别人工验证时间轴和地图展示。

## 10. 量化退出指标

- [ ] S10A-T01～S10A-T22 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 每个 ready 返回日的必要 `RouteRequirement` 覆盖率为 100%，每条恰有一个 RouteSegment；lodging_side 早餐路线数和末日返店段数均为 0。
- [ ] walking 在 1999m、2000m 时合格，在 2001m 时通过数为 0；accepted 更严格上限外的 walking/transit 接驳通过数为 0。
- [ ] transit 中缺失、负值或超过有效上限的步行接驳段被选择数为 0；public_transit/mixed 生成 self_drive 段数为 0。
- [ ] 所有固定偏好回退均按定义顺序查询；最终并列选择偏离 `(duration_seconds, distance_meters, route_id)` 的次数为 0。
- [ ] 合法空路线或全部硬约束淘汰时 `required_route_unavailable` issue 数等于缺失必要段数；直线、默认北京坐标、LLM 生成路线和部分成功路线外泄次数均为 0。
- [ ] 单请求内相同路线查询重复外部调用数为 0；每段 transit 刷新次数最大值为 1，第二次刷新成功数为 0。
- [ ] Provider/解析技术错误伪装为 `needs_repair` 的次数为 0；needs_repair 伪装最终 partial/failed 的次数为 0。
- [ ] 默认测试公网、真实高德和真实 LLM 调用数均为 0；日志/错误/证据中 Key、地址、完整坐标、polyline、payload、自由文本或完整草案命中数为 0。
- [ ] Stage 1～9B 回归与前端构建通过，新增未追踪 TODO 数为 0；03 Stage 10A 的每条退出条件及 02 的路线真实性/交通回退规则均映射到测试或证据 ID。

## 11. 迁移与回滚

- Stage 10A 只在显式新链路内部新增 RouteService；默认 `TRIP_PLANNER_MODE=legacy`，无用户数据、缓存或 API schema 迁移。
- 回滚顺序：先移除 TripCoordinator 的 RouteService 调用和内部结果保存，再移除 RouteService/RoutePolicy、模型增量与定向测试；保留 Stage 1～4 Provider/AmapService 真实路线能力及 Stage 9B 完整草案。
- 若 Stage 11 及以后已依赖本结果，必须先按依赖逆序回滚 Stage 13D→11，再回滚 Stage 10A；不得单独删除 RouteSegment 语义或用旧直线替代。
- 回滚后执行 Stage 1～9B 全量测试、legacy/shadow/new 模式回归和前端生产构建；期望退出码均为 0，普通 legacy 请求仍不加载 Coordinator。回滚不得启用虚构路线、北京默认坐标或旧 Agent fallback 作为替代成功路径。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_10a/test-summary.md` | 提交标识、三条命令、22 个用例结果、离线声明和失败数 |
| 固定方式矩阵 | `doc/opt_strategy/evidence/stage_10a/fixed-mode-matrix.md` | 三种偏好、查询顺序、实际方式映射、禁止方式与测试 ID |
| 端点/路线矩阵 | `doc/opt_strategy/evidence/stage_10a/route-requirement-matrix.md` | 每日酒店/早餐/活动/末日边界、requirement、route ID 和测试 ID |
| 约束与选择矩阵 | `doc/opt_strategy/evidence/stage_10a/filter-selection-matrix.md` | 2000m/accepted 上限、transit 接驳、淘汰原因、稳定排序和测试 ID |
| 查询/刷新矩阵 | `doc/opt_strategy/evidence/stage_10a/transit-query-refresh-matrix.md` | 预计时刻、实际偏差入口、0/1/2 次刷新和调用次数 |
| 错误与边界矩阵 | `doc/opt_strategy/evidence/stage_10a/error-boundary-matrix.md` | 技术错误、合法空路线、needs_repair、禁止直线/partial/TripPlan 与测试 ID |
| 回滚验证 | `doc/opt_strategy/evidence/stage_10a/rollback-check.md` | Coordinator 恢复点、依赖逆序、命令、结果和无数据影响 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 4、9B 实际 Done（含其 Stage 5 上游链），并在开工前复核 Stage 4 `SourcedRoute`/公交接驳字段、Stage 5 移动约束、Stage 9B 完整活动顺序及 Stage 11 公交刷新调用边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
