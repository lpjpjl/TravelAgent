# Stage 10B：混合交通真实候选选择

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 10B |
| 目标 | 仅对 `mixed` 交通偏好，为每条必要路段同步取得真实步行、公共交通和打车候选，先应用不可放宽的移动约束，再由受限 `RouteSelection` 选择已有 route ID；所有合格候选缺失时显式返回修复输入 |
| 对应优化项 | P0-6 RouteService 基础版（混合方式）、P0-7 行程深度关联化，并为 P0-8/P1-4 提供真实路线输入 |
| 前置依赖 | Stage 10A |
| 后续阻断 | 未通过时不得开始 Stage 11、12A、12B、13A～13D、16、18、20、28 |
| 默认流量影响 | 无；只扩展显式 Coordinator 新链路验证。legacy/shadow 普通请求不运行 Coordinator，`new` 仍为 reserved |

## 2. 当前代码基线与差异

- 当前 legacy 路线入口只按地址调用单一 MCP 工具并返回空字典；既没有混合交通候选集，也没有 route ID 白名单或约束过滤。
- Stage 4 提供按坐标查询 `walking/driving/transit` 的真实 `SourcedRoute` 事实；它不将 driving 映射为混合语义中的 taxi，也不删除步行超限、公交接驳超限或来源无法证明的候选。
- Stage 10A 已规定必要端点、`lodging_side` 早餐排除、末日不返店、公交查询参考与一次刷新接口，并为固定偏好以确定性方式选择路线；它明确不查询三方式候选、不调用 RouteSelection Agent。
- 当前没有 RouteSelection schema、候选 ID→路段范围对账、Agent 有限重试、Agent 明确放弃时的确定性选择，或“所有合格候选为空”与核心 Agent 技术错误的分流。
- Stage 11 才使用最终选择的真实段生成时间轴，并在实际公交出发与查询参考相差超过 30 分钟时调用 10A 的刷新接口；本 Stage 不生成最终时刻或刷新循环。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|------|------|---------------|----------|
| 修改 | `backend/app/services/route_service.py` | 复用 10A 端点展开，按每条必要段固定顺序同步查询 walking、transit、driving 三类真实候选，并复用公交参考/刷新能力 | 每条 mixed 段恰好按三种 Provider 查询方式查询一次；不选择、不导入 Agent |
| 修改 | `backend/app/services/route_policy.py` | 作为 RouteSelection 前的 ConstraintPolicy，过滤 walking 总距离与 transit 步行接驳上限，映射 driving→taxi，并提供 Coordinator 放弃时的稳定选择 | 不生成/修改来源路线；mixed 永不产生 self_drive |
| 修改 | `backend/app/models/route.py` | 增加 `MixedRouteCandidate`、过滤诊断、`RouteSelection` 判别项及候选/选择对账模型 | 可表达 selected/abstain、route ID 白名单与每段唯一选择；禁止距离/耗时等 Agent 改写字段 |
| 修改 | `backend/app/agents/itinerary_planner_agent.py` | 增加独立 `select_mixed_routes()` Prompt，只接收已过滤候选并输出 RouteSelection | 无 MCPTool/AmapService；temperature=0.3、JSON mode |
| 新增 | `backend/app/services/route_selection_service.py` | 构建脱敏候选快照、调用 StructuredLLMAdapter、校验 RouteSelection 并执行有限重试 | 成功选择只引用同段现存 route ID；不访问外部路线服务 |
| 新增 | `backend/app/services/route_selection_errors.py` | 定义 RouteSelection 校验、重试耗尽和上游候选状态的稳定错误码 | 解析/LLM 耗尽保持技术错误，不转换无路线 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | `mixed` 时编排 RouteService→RoutePolicy→RouteSelectionService；将 Agent 选择或确定性放弃回退映射为 RoutePlanningResult | 任一段无合格候选时不调用 Agent；不组装 TripPlan |
| 修改 | `backend/tests/unit/services/test_route_service.py` | 覆盖三方式串行查询、查询参考和请求内去重 | 默认公网请求数为 0 |
| 修改 | `backend/tests/unit/services/test_route_policy.py` | 覆盖混合候选过滤、来源子类和放弃后的稳定选择 | 2km 与 accepted 约束边界可离线判定 |
| 修改 | `backend/tests/unit/models/test_route_timeline.py` | 覆盖候选、RouteSelection 判别联合与禁止字段序列化 | selected/abstain 和 route ID 作用域严格校验 |
| 新增 | `backend/tests/contract/test_route_selection_agent.py` | JSON mode、temperature、Prompt 边界、白名单、重试与耗尽测试 | 不使用真实 LLM 或完整事实 payload |
| 新增 | `backend/tests/unit/services/test_route_selection_service.py` | 覆盖输入快照、逐段对账、无效选择、abstain 回退和错误分流 | 候选集不可被 Agent 改写 |
| 新增 | `backend/tests/integration/test_trip_coordinator_mixed_routes.py` | 覆盖 Stage 10A→10B 接入、候选缺失、模式与下游边界 | Coordinator 为 Stage 11 保存唯一最终 RouteSegment 集合 |

本 Stage 不修改 AmapService、Provider/解析器、Stage 9A 住宿餐饮选择、PrimaryDraft/ItineraryDraft 内容、TimelineCalculator、BudgetCalculator、API 路由、legacy Agent、前端类型或页面。RouteSelection 不会替换餐厅、酒店、活动顺序或日期；这些修复分别属于 Stage 13B/13C/13D。

## 4. 输入、输出与错误契约

### 4.1 调用前置与输入快照

TripCoordinator 仅在 `TransportationPreference=mixed`、Stage 10A 的端点展开成功、且 Stage 9B 的完整 `ItineraryDraft`、真实端点坐标、城市/adcode、accepted mobility constraints 均可用时进入本 Stage：

- Stage 9B 上游的 business_infeasible 或技术错误，以及复用 10A 端点展开时发现的技术校验错误一律原样短路；RouteService、RouteSelectionService 和 LLM 调用次数均为 0。10A 的固定方式 `needs_repair` 不属于 mixed 分支输入；
- 复用 Stage 10A 的 `RouteRequirement`、`RouteQueryReference`、`effective_walking_limit_meters`、坐标合法性、`lodging_side` 排除、非末日返店与末日无返店语义；不得重新解释端点或调用地址地理编码；
- 输入快照仅含 request_id、日期/段 ID、origin/destination 稳定 ID、有效步行上限、Provider 查询方式、候选 route ID、业务方式、来源交通子类、距离、耗时、换乘数、来源策略、polyline 是否可用及公交查询参考；不含地址、完整坐标/polyline、自由文本、完整草案、完整 payload、Prompt 或 Key；
- `walking`、`public_transit`、`self_drive` 继续使用 Stage 10A。除 `mixed` 外的值不得进入 RouteSelectionService；非法枚举为输入校验错误。

### 4.2 三方式候选查询

RouteService 对每条 `RouteRequirement` 严格按 `walking → transit → driving` 同步串行调用 Stage 4 `AmapService.plan_routes()`；每种 Provider 查询方式最多一次，合法空集合仅表示该方式无候选，不中断后续查询。

- walking 来源路线映射为业务方式 `walking`；driving 来源路线在 mixed 语义中只能映射为 `taxi`，绝不生成 `self_drive`；transit 保持业务方式 `transit`。
- transit 仅在来源事实明确时细分 `metro`、`bus` 或 `tourist_shuttle`；来源未提供足够事实时为受控 `unknown`。不得根据线路名称、城市常识或 Agent 理由生成旅游专线大巴。
- 同一 Provider 查询方式返回的全部合法 `SourcedRoute` 都进入后续过滤；不得先按最短路线压缩候选、合并不同 route ID、捏造路线或用直线替代。
- 为使逐段 transit 查询时刻可复现，RouteService 在每条前序段三方式查询、过滤后，临时按 `(duration_seconds, distance_meters, route_id)` 选择一个仅用于推进下一段预计时钟的 provisional candidate。它不是最终选择、不输出给 Agent/前端；后续 RouteSelection 选择不同 transit 段时仍由 Stage 11 依 10A 的一次刷新规则处理。
- 相同端点、Provider 查询方式、策略和 transit 查询参考在同一请求内必须复用查询结果；本 Stage 不引入并发、缓存、Provider 自动切换或跨请求复用。

### 4.3 ConstraintPolicy 过滤与候选集

`RoutePolicy` 在 RouteSelection 前承担本 Stage的 ConstraintPolicy 职责。它以 `effective_walking_limit_meters = min(2000, accepted mobility constraint)` 过滤候选，边界为闭区间：

| 候选 | 合格规则 | 典型过滤 reason |
|------|----------|------------------|
| `walking` | 来源 `distance_meters <= effective_walking_limit_meters` | `walking_distance_exceeds_limit` |
| `transit` | 每个来源实际步行接驳段距离均存在、非负且 `<= effective_walking_limit_meters` | `transit_walking_distance_unavailable` / `transit_walking_segment_exceeds_limit` |
| `taxi` | driving 来源路线具备 Stage 4 所需核心事实 | 不因点位距离或 2km 步行规则过滤 |

- 被过滤候选只保留脱敏 route ID、方式、稳定 reason 与计数用于日志/证据，不进入 Agent 快照；过滤不能修改来源距离、耗时、polyline、路线子类或 data quality。
- 每个合格 `MixedRouteCandidate` 必须包含 route ID、requirement ID、实际方式、Provider 查询方式、来源 transit 子类、距离、耗时、换乘事实、polyline、来源/策略与查询参考；route ID 在同一路段内唯一。不同路段可存在相同 route ID，但 RouteSelection 不得跨段引用。
- 如任一必要段过滤后候选集为空，Coordinator 立即产生该段 `RoutePlanningIssue(error_code="required_route_unavailable")`；不调用 Agent、不返回部分 RouteSegment、不以“超过 2km”直接判整日 failed，且后续由 Stage 13B/13D 决定修复或最终状态。

### 4.4 RouteSelection Agent 契约

当且仅当每个必要段至少有一个合格候选时，RouteSelectionService 复用 Stage 5 `StructuredLLMAdapter` 调用 `ItineraryPlannerAgent.select_mixed_routes()`：

- 单次 `RouteSelection` 覆盖本次所有 requirement，`selections` 必须按 `day_index/leg_index` 升序、每段恰好一项；Agent 不能改变段、端点、候选集合或活动顺序。
- 每项为严格判别联合：`selected` 时仅输出该段输入白名单中的 `route_id` 与 1～160 个 Unicode code point 的选择理由；`abstain` 时 `route_id=null` 并输出受控 reason。`abstain` 是“Agent 无有效偏好”的唯一合法表达，不能夹带自造 route ID。
- Agent 只能选择已有候选或 abstain；不得输出/改写距离、耗时、换乘、交通子类、polyline、坐标、地址、预计/最终时刻、费用、预算、过滤原因、路线策略、TripPlan 或新的事实实体。
- 选择理由只解释候选间便利性，不得声称未在输入中提供的价格、营业、拥堵、实时班次、无障碍、天气或可执行性事实；理由属于 generated 决策数据，不改变路线事实。
- 调用固定为 JSON mode/`response_format`、`temperature=0.3`。响应先 `json.loads` 再以 `extra="forbid"` 模型校验，禁止正则提取、代码块修复或静默删除字段。

### 4.5 校验、有限重试与确定性放弃回退

每次 Agent 响应依次校验：

1. JSON、`extra="forbid"`、request_id 与 requirement 数量/排序；
2. 每个 requirement 恰好一次，未知/重复/遗漏/跨段 route ID 全部拒绝；
3. `selected` 的 route ID 必须在该段合格候选白名单内；`abstain` 必须为 null route ID 和受控 reason；
4. 输出没有被禁止的事实、时间、费用、地图或 TripPlan 字段；
5. selected reason 长度与脱敏规则通过，且 ignored requirements、天气和用户原文未进入输入/输出。

- JSON/schema/白名单/对账/禁止字段错误可重试，首次失败后最多重试 2 次，总调用上限为 3；每次使用完全相同的候选快照、schema 和 temperature，仅追加稳定脱敏校验码。
- 三次仍无合法 RouteSelection，或 LLM 认证、超时、限流等技术异常耗尽时抛 `RouteSelectionError(error_code="route_selection_retry_exhausted")`。这是核心规划技术错误，绝不伪装为 abstain、needs_repair、ignored、partial 或 fallback。
- 合法 `abstain` 不重试。Coordinator 仅对该段在合格候选中按 `(duration_seconds, distance_meters, route_id)` 选择一条，并记录 `selection_origin=coordinator_fallback`；`selected` 记录 `selection_origin=agent`。两种结果都映射为真实 `RouteSegment`，不得改变来源事实。
- Agent 返回非空有效 route ID 但选择了次优候选是合法业务决策；Coordinator 不以最短耗时覆盖 Agent 选择。确定性排序只用于 abstain，不构成价格、舒适度、换乘惩罚或高级多目标策略。

### 4.6 输出、错误与日志

协调结果沿用 Stage 10A `RoutePlanningResult`：

| 状态 | 输出 | 语义 |
|------|------|------|
| `ready` | 每个 requirement 恰有一个 RouteSegment，按 day/leg 排序；每段含真实 route ID、实际方式、来源事实、查询参考和 `selection_origin` | 所有必要段均从本次合格候选中选择 |
| `needs_repair` | 无 RouteSegment；仅含稳定排序的 `required_route_unavailable` issues | 至少一个必要段三种方式均无合格来源路线 |

- ProviderError、AmapParseError、输入错误、RouteSelection JSON/LLM 耗尽和 10A 的 transit refresh limit 都保持技术/校验错误，不能转换为 `needs_repair`；只有三方式已完成且合格候选为空才形成 `required_route_unavailable`。
- 日志记录 request_id、stage=`routes_mixed_query|route_selection`、day/leg、Provider 查询方式、合格/过滤计数与稳定 reason、候选 route ID、Agent attempt、selected/abstain、selection_origin、耗时和稳定错误码；不得记录 Key、地址、完整坐标/polyline、完整 payload、自由文本、完整 Prompt 或完整 Agent 输出。

## 5. 配置与功能开关

- 本 Stage 不新增 Settings、环境变量或用户流量开关。`MAX_WALKING_SEGMENT_METERS=2000` 与 accepted 移动约束的“只能收紧”语义沿用 Stage 2/5/10A。
- 沿用 `AMAP_PROVIDER`、`AMAP_API_KEY`、LLM 配置、StructuredLLMAdapter 注入和 `TRIP_PLANNER_MODE`；RouteService、RoutePolicy、RouteSelectionService 与 Agent 不直接读取 Settings。
- JSON mode、temperature=0.3、最多 2 次重试、三种 Provider 查询顺序、abstain 回退排序和每段 transit 最多一次刷新均为受测试锁定的领域规则，不提供环境变量覆盖。
- legacy/shadow 普通请求不执行本 Stage；显式 shadow 验证可执行内部结果；new 在 Stage 16 前仍为 `planner_mode_not_ready`。

## 6. 实施步骤

1. 扩展 `models/route.py`，定义 mixed candidate、来源 transit 子类、过滤诊断、RouteSelection selected/abstain 联合、候选作用域和 `selection_origin` 的严格模型与正反例。
2. 扩展 RouteService，复用 10A 端点/预计时刻/刷新语义，为每个 requirement 串行查询 walking、transit、driving 的全部真实来源候选，并以 provisional candidate 推进仅供查询使用的时钟。
3. 扩展 RoutePolicy 的 ConstraintPolicy 职责，确定性过滤步行总距离和 transit 每段接驳距离，映射 driving→taxi；不在此处选择 Agent 路线或生成事实。
4. 建立 RouteSelectionService 与 ItineraryPlannerAgent 独立 Prompt，固定候选快照、JSON mode、temperature、ID 白名单、禁止字段、selected/abstain 和两次重试边界。
5. 实现候选/段全覆盖校验、核心技术错误 `route_selection_retry_exhausted`，以及仅对合法 abstain 的 Coordinator 最短耗时确定性回退。
6. 在 TripCoordinator 的 mixed 分支接入三方式查询→过滤→RouteSelection→RouteSegment 映射；任何空候选段短路为 needs_repair，禁止调用 Agent 或输出部分路线。
7. 使用 FakeAmapProvider 与 FakeLLM 运行定向、全量离线回归和前端构建；归档候选、过滤、选择、重试、错误和回滚证据。

## 7. 明确不做

- 不改变 Stage 10A 的 walking/public_transit/self_drive 固定方式查询、回退或选择语义。
- 不查询或输出 self_drive 候选，不根据名称臆造旅游专线，不计算价格、换乘惩罚、舒适度、天气影响或其他高级多目标评分；后者属于 Future F-1。
- 不生成最终时刻、餐窗、wait、buffer、时间冲突或触发公交刷新循环；属于 Stage 11/12B。
- 不替换路线之外的餐厅/酒店、重排/删点、生成 partial/failed，或组装/返回 TripPlan；属于 Stage 13A～13D/16。
- 不缓存、并发、Provider 自动切换、地图展示、API schema 变更、SSE 或前端改造；分别属于 Stage 17、20、28 等后续 Stage。
- 不使用地址地理编码、默认城市坐标、直线距离、LLM 生成路线或部分成功段替代必要真实路线。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S10B-T01 | unit | 1～3 日 RouteRequirement | 完全复用 10A 端点顺序、lodging_side 排除和末日无返店 |
| S10B-T02 | unit | 每段三方式查询 | walking→transit→driving 串行各一次；合法空不阻断后续方式 |
| S10B-T03 | unit | driving/transit 来源映射 | driving 仅为 taxi；metro/bus/tourist_shuttle 只保留来源明确事实，未知不猜测 |
| S10B-T04 | unit | 1999m/2000m/2001m walking | 前两者保留，2001m 过滤但 taxi/transit 候选继续存在 |
| S10B-T05 | unit | accepted 1200m 与 transit 接驳 | walking/每个接驳均按更严格上限过滤；缺失/负接驳距离过滤 |
| S10B-T06 | unit | taxi 3km/10km | taxi 不因 2km walking 规则过滤 |
| S10B-T07 | unit | 多 route ID 与请求内去重 | 同一方式全部合法候选保留；相同查询仅外部调用一次 |
| S10B-T08 | unit | provisional transit 查询时钟 | 仅按稳定 provisional candidate 推进；不泄漏为最终选择 |
| S10B-T09 | unit | 单段/多段全候选过滤为空 | 无 LLM 调用，输出稳定排序 required_route_unavailable issues，无部分 segments |
| S10B-T10 | contract | 合法 RouteSelection | JSON mode、temperature=0.3、extra=forbid；每段唯一 selected route ID |
| S10B-T11 | contract | selected route ID 未知、跨段、重复、遗漏或无序 | 全部拒绝并重试 |
| S10B-T12 | contract | Agent 改写距离/耗时/换乘/polyline/时刻/费用/TripPlan | schema 拒绝并重试 |
| S10B-T13 | contract | 合法 abstain | route_id=null 且受控 reason；不重试，交由 Coordinator 回退 |
| S10B-T14 | unit | abstain 确定性回退 | 按 `(duration_seconds, distance_meters, route_id)` 选择，标记 coordinator_fallback |
| S10B-T15 | unit | Agent 合法次优选择 | 保留 Agent route ID，不被 Coordinator 最短耗时覆盖 |
| S10B-T16 | contract | 首次成功、第三次成功、三次非法与技术异常耗尽 | 调用数分别为 1、3、3、3；耗尽 route_selection_retry_exhausted |
| S10B-T17 | unit | 候选/选择模型 | selected/abstain 判别、route ID 作用域、禁止 self_drive 和非来源旅游专线均严格拒绝 |
| S10B-T18 | unit | ProviderError/AmapParseError | 原样传播，不转换 needs_repair 或 abstain |
| S10B-T19 | integration | Stage 10A→10B happy path | Coordinator 保存完整 mixed RouteSegment 集合，供 Stage 11 使用 |
| S10B-T20 | integration | 上游短路与全候选缺失 | 前者外部/LLM 调用为 0；后者 LLM 为 0 且只保存修复输入 |
| S10B-T21 | regression | fixed/mixed/legacy/shadow/new 模式 | fixed 保持 10A；legacy/shadow 普通请求调用数为 0；new 仍 reserved |
| S10B-T22 | regression | Stage 1～10A 与前端 | 上游离线测试和生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/models/test_route_timeline.py backend/tests/unit/services/test_route_service.py backend/tests/unit/services/test_route_policy.py backend/tests/unit/services/test_route_selection_service.py backend/tests/contract/test_route_selection_agent.py backend/tests/integration/test_trip_coordinator_mixed_routes.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、真实高德或真实 LLM。Stage 10B 必测用例数为 22，失败数必须为 0。

## 9. 人工验收

不适用：本 Stage 不接入普通 API/UI，也不得为验收消耗真实路线或 LLM 配额。三方式查询、来源子类、候选过滤、Agent 选择、abstain 回退、错误和调用次数均由 FakeAmapProvider、FakeLLM 与固定 fixture 自动化覆盖；Stage 11/20 才分别验证时间轴和地图呈现。

## 10. 量化退出指标

- [ ] S10B-T01～S10B-T22 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] mixed 每个必要段的 walking/transit/driving Provider 查询顺序正确率为 100%，同一查询重复外部调用数为 0；mixed 生成 self_drive 段数为 0。
- [ ] 1999m/2000m walking 保留率为 100%，2001m walking 被选中数为 0；taxi 因 2km 规则被过滤数为 0。
- [ ] transit 中缺失、负值或超有效上限的步行接驳候选被输入 Agent/被选择数均为 0；非来源明确的 tourist_shuttle 生成数为 0。
- [ ] Agent 输入 route ID 全部来自当段合格候选；跨段、未知、重复或遗漏 ID 的接受数为 0；Agent 改写事实字段的接受数为 0。
- [ ] 每个 ready requirement 的选择覆盖率为 100%；合法 abstain 的 Coordinator 回退排序偏离 `(duration_seconds, distance_meters, route_id)` 的次数为 0。
- [ ] 任一空候选段的 LLM 调用数和部分 RouteSegment 外泄数均为 0；`required_route_unavailable` issues 与缺失段数量一致。
- [ ] 三次 RouteSelection 非法/技术失败均为 `route_selection_retry_exhausted` 技术错误，伪装为 abstain/needs_repair/ignored/partial/fallback 的次数为 0。
- [ ] 默认测试公网、真实高德和真实 LLM 调用数为 0；日志、错误、fixture 和证据中 Key、地址、完整坐标/polyline、完整 payload、自由文本、完整 Prompt 或完整 Agent 输出命中数为 0。
- [ ] Stage 1～10A 回归与前端构建通过，新增未追踪 TODO 数为 0；03 Stage 10B 的每条退出条件及 02 的混合交通真实性/约束/回退规则均映射到测试或证据 ID。

## 11. 迁移与回滚

- Stage 10B 只为显式新链路增加 mixed 分支，默认 `TRIP_PLANNER_MODE=legacy`；无用户数据、缓存、API schema 或前端迁移。
- 回滚时先移除 TripCoordinator 的 mixed RouteSelection 分支，再移除 RouteSelectionService、错误模型、Agent 第三调用入口和 mixed 模型增量；恢复 Stage 10A 固定方式能力，但不得把 mixed 请求静默降级为固定方式。
- 已有 Stage 11 及以后依赖时，必须先按依赖逆序回滚 Stage 13D→11，再回滚 Stage 10B；不得删除 10A 的 RouteRequirement、RouteQueryReference 或真实路线语义。
- 回滚后执行 Stage 1～10A 全量测试、fixed/mixed/legacy/shadow/new 模式回归和前端生产构建，期望退出码均为 0。回滚不得启用直线、默认北京坐标、虚构路线、虚构 fallback 或未过滤 transit 候选。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_10b/test-summary.md` | 提交标识、三条命令、22 个用例结果、离线声明和失败数 |
| 查询/候选矩阵 | `doc/opt_strategy/evidence/stage_10b/mixed-query-candidate-matrix.md` | requirement、三方式调用顺序、route ID、来源方式/子类与测试 ID |
| 过滤矩阵 | `doc/opt_strategy/evidence/stage_10b/constraint-filter-matrix.md` | 2km/accepted 上限、transit 接驳、taxi 例外、过滤 reason 与测试 ID |
| RouteSelection 契约 | `doc/opt_strategy/evidence/stage_10b/route-selection-contract.md` | 输入白名单、selected/abstain、禁止字段、重试与测试 ID |
| 选择/回退矩阵 | `doc/opt_strategy/evidence/stage_10b/selection-fallback-matrix.md` | Agent 选择、abstain 稳定排序、selection_origin 和测试 ID |
| 错误与边界矩阵 | `doc/opt_strategy/evidence/stage_10b/error-boundary-matrix.md` | 空候选、技术错误、retry exhausted、禁止 partial/直线与测试 ID |
| 回滚验证 | `doc/opt_strategy/evidence/stage_10b/rollback-check.md` | mixed 分支撤销、依赖逆序、命令、结果和无数据影响 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 10A 实际 Done，并在开工前复核 RouteRequirement/RouteQueryReference、ConstraintPolicy 过滤语义、Stage 4 transit 接驳事实以及 Stage 11 对一次刷新接口的消费边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
