# Stage 9A：连续住宿与便餐候选/筛选

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 9A |
| 目标 | 在 `PrimaryItineraryDraft` 已确定后，选择连续住宿记录并为未被美食级餐点占用的午晚餐槽位补齐真实便餐 POI，同时生成早餐安排 |
| 对应优化项 | P0-5 餐饮候选检索与筛选、P0-7 行程深度关联化 |
| 前置依赖 | Stage 8 |
| 后续阻断 | 未通过时不得开始 Stage 9B、10A、11、12B、13A、18 |
| 默认流量影响 | 仅扩展显式新链路/bootstrap；legacy 与普通 shadow 请求不选择住宿或便餐，默认 `/trip/plan` 响应不变 |

## 2. 当前代码基线与差异

- 当前 legacy Agent 在自由文本中直接生成酒店和餐饮建议，无法证明住宿、午餐或晚餐来自本次高德真实候选，也无法保证 `lodging[0..N-1]` 连续。
- Stage 8 已输出严格 `PrimaryItineraryDraft`：每日主要点顺序、`meal_role`、三餐 provisional slots，以及 `DAY_START / LUNCH_SLOT / DAY_END / POI_ID` 锚点，但不包含酒店或便餐 ID。
- 当前没有连续住宿检索策略、住宿选择 schema、早餐 `lodging_side/gourmet` 判别、便餐候选检索、饮食排除复用、固定餐窗预筛或 `LUNCH_SLOT` 解析规则。
- Stage 9B 才负责将 PrimaryDraft、住宿和便餐合并为完整 `ItineraryDraft`；本 Stage 不能输出完整活动顺序、路线、最终时刻、费用或 TripPlan。
- Stage 10A/10B 才调用真实路线；本 Stage 只能基于锚点检索和筛选实体，不能用直线距离或估算耗时证明可执行。
- Stage 11/12B 才校验最终时间轴；本 Stage 只做午晚餐候选固定餐窗内连续 `>=40min` 的预筛，不能声明最终实际用餐区间已满足。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/services/lodging_search_service.py` | 根据 PrimaryDraft 空间锚点同步串行检索酒店候选，构建每日住宿选择输入 |
| 新增 | `backend/app/services/meal_search_service.py` | 按 lunch 优先、dinner 后续的顺序检索真实便餐 POI，执行饮食排除和营业时间候选预筛 |
| 新增 | `backend/app/services/lodging_meal_selection_service.py` | 调用 DestinationResearchAgent 选择住宿和便餐 ID，校验连续住宿、餐次、锚点和 required 语义 |
| 新增 | `backend/app/services/lodging_meal_errors.py` | 定义住宿/便餐业务不可行、候选冲突和重试耗尽等稳定错误码 |
| 修改 | `backend/app/agents/destination_research_agent.py` | 增加住宿选择和便餐选择两个结构化入口；继续只引用候选 ID，不生成事实实体 |
| 修改 | `backend/app/models/itinerary.py` | 增加 `LodgingRecord`、`BreakfastArrangement`、`ConvenienceMealSelection`、`LodgingMealPlan` 与 anchor 解析结果 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 8 PrimaryDraft 后串行执行住宿选择、早餐映射和缺失午晚餐便餐补齐 |
| 新增 | `backend/tests/unit/services/test_lodging_search_service.py` | 住宿检索锚点、数量、空结果、去重和错误传播测试 |
| 新增 | `backend/tests/unit/services/test_meal_search_service.py` | 午晚餐检索顺序、`LUNCH_SLOT` 解析、饮食过滤、餐窗预筛和空候选测试 |
| 新增 | `backend/tests/unit/services/test_lodging_meal_selection_service.py` | 住宿连续性、早餐安排、便餐选择、ID 白名单、重试和错误分流测试 |
| 新增 | `backend/tests/contract/test_destination_research_agent_lodging_meals.py` | 住宿/便餐 JSON schema、temperature、Prompt 边界、重试与耗尽测试 |
| 新增 | `backend/tests/integration/test_trip_coordinator_lodging_meals.py` | Stage 8→9A 接入、上游短路、模式回归和下游边界测试 |

本 Stage 不修改 Stage 8 PrimaryDraft 生成逻辑、AmapService parser、Provider、RouteService、TimelineCalculator、BudgetCalculator、API 路由、legacy Agent、前端类型或页面。

## 4. 输入、输出与错误契约

### 4.1 调用前置与输入快照

- 只允许 `PrimaryItineraryDraft` 合法且上游状态为可继续规划时进入本 Stage；Stage 8 business_infeasible 或技术错误由 Coordinator 原样短路。
- 输入必须包含 request_id、city、start_date、end_date、travel_days、preferences、normalized accepted constraints、PrimaryDraft、按 candidate ID 回填的真实主要点位事实，以及 Stage 5 `DietaryRuleSet`。
- `travel_days` 只允许 1～3；本 Stage 必须输出恰好 N 个 `LodgingRecord` 和 N 个 `BreakfastArrangement`。
- ignored requirements 不进入住宿或便餐检索、Agent Prompt、候选过滤或选择理由；accepted constraints 只以 normalized 字段和稳定 ID 进入。
- `weather_context` 在 Stage 18 前不得影响住宿、便餐候选检索、筛选或理由；天气导致重建属于 Stage 18 后的重新评审范围。

### 4.2 连续住宿候选检索

- `lodging[0]` 依附首日第一个主要点位分布检索；中间住宿 `lodging[i]` 同时考虑 Day i-1 最后主要点位和 Day i 第一个主要点位；`lodging[N-1]` 服务末日前一晚，不为末日晚餐后新增住宿。
- 每个住宿检索锚点必须来自 PrimaryDraft 已安排的真实主要点位 POI ID；不得使用便餐、未来酒店、路线估算点或自由文本。
- AmapService 使用固定酒店关键词和城市限制同步串行检索，每个住宿位置候选 `page_size=10`；允许同一真实酒店被多个住宿位置复用。
- 住宿候选按 `(provider, source_id)` 稳定去重；同 ID 核心身份字段冲突抛 `LodgingMealError(error_code="lodging_source_identity_conflict")`。
- 某个住宿位置合法空结果返回业务不可行 `lodging_candidate_unavailable`，不得让 Agent 编造酒店；Provider/解析错误按技术错误传播。

### 4.3 住宿选择契约

`DestinationResearchAgent.select_lodging_chain()` 使用 JSON mode/`response_format`、`temperature=0`，只能从每个位置的候选 ID 中选择酒店。输出 `LodgingRecord[]` 必须满足：

- 数量恰好等于 travel_days，索引为 `0..N-1`；
- 每项只引用真实 hotel candidate ID，包含 source key、position_index、选择理由和可用事实质量；
- 第 i 天 `start_hotel_id=lodging[i]`；当 i<N-1 时 `end_hotel_id=lodging[i+1]`，当 i=N-1 时 `end_hotel_id=null`；
- 酒店可连续复用，也可 A→B 换宿；一旦 Day i 结束酒店为 B，Day i+1 必须从同一个 B 出发；
- 输出不得包含路线、距离、最终时刻、费用、房价承诺或完整活动顺序。

住宿选择三次均失败时抛 `LodgingMealError(error_code="lodging_selection_retry_exhausted")`，属于核心规划技术错误，不得转 ignored、partial 或虚构 fallback。

### 4.4 早餐安排

每个日期必须生成唯一 `BreakfastArrangement`：

| 来源 | 条件 | 输出规则 |
|------|------|----------|
| `gourmet` | PrimaryDraft 当日 breakfast slot 已被美食级餐点占用 | 引用该日唯一 gourmet candidate ID；不再生成 `lodging_side` |
| `lodging_side` | breakfast slot 为 `requires_convenience_meal` | 引用当日 `start_hotel_id`，不生成 POI、路线或外出事件 |

- `lodging_side` 早餐不受 `dietary_exclusion` 约束，不声称系统已验证饮食属性。
- gourmet 早餐不做固定候选餐窗预筛；最终营业时间覆盖校验留给 Stage 11/12B。
- 同一天 `gourmet` 与 `lodging_side` 不得并存；早餐不得缺失。

### 4.5 便餐候选检索顺序与锚点解析

- 只为 PrimaryDraft 中 lunch/dinner slot 为 `requires_convenience_meal` 的餐次检索便餐；`occupied_by_gourmet` 的餐次不得重复检索或替换。
- 每日按午餐优先、晚餐后续同步串行处理。若 dinner `after_anchor=LUNCH_SLOT`，必须先选定当日午餐便餐并将 `LUNCH_SLOT` 解析为该真实午餐 POI ID 后，才能检索晚餐。
- `DAY_START` 解析为当日 `start_hotel_id`；`DAY_END` 在非末日解析为当日 `end_hotel_id`，末日 dinner `before_anchor=DAY_END` 仅表示旅程结束边界，不生成末日结束酒店。
- lunch 候选位于 `lunch_after_anchor` 与 `lunch_before_anchor` 之间；dinner 候选位于 `dinner_after_anchor` 之后，非末日还必须考虑 `end_hotel_id` 边界；末日晚餐作为旅程终点。
- AmapService 返回真实餐饮 POI，固定 `page_size=10`，候选必须包含具体店名/分店名、地址、坐标和来源 ID；“附近餐厅”“当地餐馆”等泛化文本不是候选实体。
- 任一必需午晚餐合法空结果返回业务不可行 `convenience_meal_candidate_unavailable`；技术错误立即停止并传播。

### 4.6 饮食排除与营业时间预筛

- 对 Stage 9A 新检索的便餐候选执行全部 accepted `dietary_exclusion` 规则；命中任一规则即排除，并记录脱敏 `MealExclusionRecord`。
- `dietary_exclusion` 不作用于 `lodging_side` 早餐；Stage 7A 已过滤过的美食级餐点不在本 Stage 重新按饮食规则恢复或改写。
- 对 Stage 8 已占用 lunch/dinner 的美食级餐点和本 Stage 新检索的便餐，若有来源营业时间，则候选阶段必须校验对应固定餐窗内存在连续 `>=40min` 可用区间。
- 固定餐窗为 lunch `12:00-14:00`、普通日晚餐 `18:00-20:00`、末日晚餐 `17:00-18:00`；每顿仍不少于 40 分钟。
- 营业时间缺失或不可解析时保留候选，标记 `unavailable` 并携带核实提示；不得补默认营业时间。
- 美食级早餐跳过固定餐窗预筛，最终实际早餐区间校验留给 Stage 11/12B。
- 因饮食或有来源营业时间预筛后候选为空时返回业务不可行，不调用 Agent 编造替代。

### 4.7 便餐选择契约

`DestinationResearchAgent.select_convenience_meal()` 使用 JSON mode/`response_format`、`temperature=0`，每次只选择一个待补午餐或晚餐。输出 `ConvenienceMealSelection` 必须满足：

- 只引用本餐次候选 ID，不能引用美食级餐点、酒店、主要景点或其他日期餐厅；
- 包含 date、meal_role=`lunch|dinner`、candidate_id、anchor snapshot、required=true、constraint IDs、选择理由和候选质量标记；
- lunch/dinner 便餐一旦选中即为 required，必须进入 Stage 9B、路线、时间轴和预算；
- selected meal 不得覆盖 PrimaryDraft 已占用的 gourmet 餐次，也不得改变 PrimaryDraft 主要点位顺序或 `meal_role`；
- 输出不得包含最终活动顺序、路线、时刻、价格、预算或 TripPlan 字段。

三次均失败时抛 `LodgingMealError(error_code="convenience_meal_selection_retry_exhausted")`，属于核心规划技术错误；合法空候选、过滤后无候选和 accepted 约束不可满足属于业务不可行。

### 4.8 输出模型

`LodgingMealPlan` 至少包含：

- request_id、city、start_date、end_date、travel_days；
- `lodging`：长度为 N 的 `LodgingRecord[]`；
- `day_boundaries`：每一天的 `start_hotel_id`、非末日 `end_hotel_id`、末日 `end_hotel_id=null`；
- `breakfast_arrangements`：长度为 N，每日唯一；
- `convenience_meals`：只包含需要补齐的 lunch/dinner 真实 POI；
- `meal_slot_resolution`：每个日期 breakfast/lunch/dinner 的最终来源为 `gourmet|lodging_side|convenience_meal`；
- `search_records` 与 `exclusions`：记录住宿、便餐检索、饮食过滤、营业时间预筛和空结果诊断；
- 派生计数：lodging_count、breakfast_count、convenience_lunch_count、convenience_dinner_count、gourmet_meal_count。

输出不包含完整 `ItineraryDraft`、路线、最终时刻、缓冲、预算、地图 polyline、前端展示字段或 TripPlan。

### 4.9 技术错误、业务不可行与日志

- request_id/city 不一致、未知 ID、跨日 anchor、`LUNCH_SLOT` 未解析即检索晚餐、住宿连续性破坏、同餐次重复、模型派生计数不一致属于技术/契约错误。
- 住宿候选合法空、便餐候选合法空、饮食过滤后无可用候选、有来源营业时间不满足固定餐窗、accepted 硬约束无法满足属于业务不可行。
- LLM 输出非法且重试耗尽属于核心规划技术错误，不得转换为 ignored、partial 或 fallback。
- 日志按 request_id 记录 stage、attempt、day、meal_role、candidate count、selected count、filter count、status、reason/error code 和 duration；不得记录 Key、Prompt、自由文本、完整 POI、完整上游 payload 或完整 Agent 输出。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或功能开关；酒店/餐饮 `page_size=10`、DestinationResearchAgent temperature=0、最多重试 2 次、午晚餐固定餐窗和 `>=40min` 预筛是受测试锁定的领域策略。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求不运行 Coordinator；显式 shadow 验证可执行 Stage 9A；new 仍为 reserved。
- AmapService、DestinationResearchAgent、StructuredLLMAdapter、DietaryRuleSet 和时钟由装配层注入；Stage 9A 服务不读取 Settings，不访问网络以外的隐式全局状态。

## 6. 实施步骤

1. 定义 `LodgingRecord`、`BreakfastArrangement`、`ConvenienceMealSelection`、`MealSlotResolution`、`LodgingMealPlan` 和稳定错误码，补序列化正反例。
2. 实现住宿检索锚点构建：首日依附第一个主要点，中间住宿依附前一日最后点和后一日第一点，末日前一晚依附末日主要点。
3. 实现住宿候选去重、空结果、事实冲突和 Agent 住宿选择，对账 N 个 `lodging` 与每日起止酒店连续性。
4. 将早餐槽位确定性映射为 `gourmet` 或 `lodging_side`，保证每日唯一且不伪造早餐 POI。
5. 实现 lunch 优先的便餐检索，解析 `DAY_START/DAY_END` 为住宿边界，保留 `LUNCH_SLOT` 正反例。
6. 实现便餐饮食排除和午晚餐固定餐窗预筛；营业时间缺失保留并标记 `unavailable`。
7. 实现 dinner 后续处理：已有 gourmet lunch 用该 POI，待选午餐用解析后的真实午餐 POI，非末日结合 `end_hotel_id`，末日晚餐作为旅程终点。
8. 实现便餐 Agent 选择、ID 白名单、required 语义、重试耗尽和业务不可行分流。
9. 接入 TripCoordinator，保存 `LodgingMealPlan` 供 Stage 9B 使用；保持 legacy/shadow/new 模式行为不变。
10. 执行 Stage 1～9A 定向测试、后端全量回归和前端构建，归档住宿、餐次、过滤、错误和回滚证据。

## 7. 明确不做

- 不修改 PrimaryDraft，不改变主要点位日期、顺序、`day_part`、`meal_role`、required 或 removal rank。
- 不生成完整 `ItineraryDraft`，不决定全部活动顺序；属于 Stage 9B。
- 不调用 RouteService，不计算真实路线、耗时、缓冲、最终时刻或可达性；属于 Stage 10A～12B。
- 不计算住宿、餐饮、交通或门票预算；属于 Stage 12A。
- 不为早餐检索或伪造 POI；非 gourmet 早餐只能是 `lodging_side`。
- 不对美食级早餐做固定餐窗预筛；不补默认营业时间或默认坐标。
- 不让天气影响住宿或便餐选择；Stage 18 接入天气后必须重新评审重建边界。
- 不修改普通 API、前端、legacy 链路，不开放 new 模式，不删除 legacy fallback。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S9A-T01 | unit | 1～3 日住宿锚点构建 | `lodging[0..N-1]` 检索位置数量正确，锚点均来自 PrimaryDraft 真实主要点 |
| S9A-T02 | unit | 单日住宿 | 输出 1 个 lodging，Day1 start_hotel 为 lodging[0]，end_hotel=null |
| S9A-T03 | unit | 多日 A→B 换宿 | Day i end_hotel 等于 Day i+1 start_hotel |
| S9A-T04 | unit | 住宿候选合法空 | business_infeasible/lodging_candidate_unavailable，不调用住宿 Agent |
| S9A-T05 | unit | 住宿 source ID 核心冲突 | 抛 lodging_source_identity_conflict |
| S9A-T06 | contract | 住宿 Agent 合法选择 | JSON mode、temperature=0、只引用候选 ID |
| S9A-T07 | contract | 住宿 Agent 输出未知/重复/越界位置 | 拒绝并重试，耗尽为 lodging_selection_retry_exhausted |
| S9A-T08 | unit | gourmet 早餐 | BreakfastArrangement 使用 gourmet ID，无 lodging_side |
| S9A-T09 | unit | 非 gourmet 早餐 | BreakfastArrangement 使用 lodging_side 和 start_hotel_id，不生成 POI |
| S9A-T10 | unit | 早餐并存或缺失 | 均拒绝 |
| S9A-T11 | unit | 已占用 lunch/dinner gourmet | 不检索同餐次便餐 |
| S9A-T12 | unit | lunch DAY_START/DAY_END 解析 | 分别解析为 start_hotel 和非末日 end_hotel/末日边界 |
| S9A-T13 | unit | dinner 无下午点且 LUNCH_SLOT | 先选午餐，再以真实午餐 POI 作为 dinner after_anchor |
| S9A-T14 | unit | dinner 已有 gourmet lunch | dinner after_anchor 使用 gourmet lunch POI，不使用 LUNCH_SLOT |
| S9A-T15 | unit | LUNCH_SLOT 未解析就检索晚餐 | 技术错误，晚餐 AmapService 调用数为 0 |
| S9A-T16 | unit | 便餐候选泛化名称或无 source ID | 拒绝候选，不进入 Agent |
| S9A-T17 | unit | dietary_exclusion 命中便餐 | 排除并记录 rule IDs/constraint IDs |
| S9A-T18 | unit | lodging_side 早餐 + dietary_exclusion | 不过滤住宿侧早餐，不声称已验证饮食 |
| S9A-T19 | unit | lunch 固定餐窗 | 有来源营业时间必须存在 12:00-14:00 内连续 >=40min |
| S9A-T20 | unit | 普通日晚餐固定餐窗 | 有来源营业时间必须存在 18:00-20:00 内连续 >=40min |
| S9A-T21 | unit | 末日晚餐固定餐窗 | 有来源营业时间必须存在 17:00-18:00 内连续 >=40min |
| S9A-T22 | unit | 营业时间缺失 | 保留候选，标记 unavailable 和核实提示 |
| S9A-T23 | unit | 过滤后无便餐候选 | business_infeasible/convenience_meal_candidate_unavailable，便餐 Agent 调用数为 0 |
| S9A-T24 | contract | 便餐 Agent 合法选择 | 只引用本餐次候选 ID，required=true，不输出路线/时刻/费用 |
| S9A-T25 | contract | 便餐 Agent 非法输出耗尽 | convenience_meal_selection_retry_exhausted，不返回部分选择 |
| S9A-T26 | unit | 每个午晚餐槽位解析 | occupied gourmet 或 convenience meal 恰好一个真实 POI |
| S9A-T27 | unit | meal_slot_resolution 派生计数 | breakfast/lunch/dinner 数量与 N 日槽位一致 |
| S9A-T28 | integration | Stage 8→9A happy path | Coordinator 保存 LodgingMealPlan，供 Stage 9B 使用 |
| S9A-T29 | integration | Stage 8 business_infeasible | 住宿和便餐调用数均为 0，状态原样短路 |
| S9A-T30 | integration | ignored 和天气变化 | Stage 18 前住宿/便餐输入与选择一致 |
| S9A-T31 | regression | legacy/shadow/new 模式 | 普通 legacy/shadow 不执行 Stage 9A，new 仍 planner_mode_not_ready |
| S9A-T32 | regression | Stage 1～8 与前端 | 上游测试和生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_lodging_search_service.py backend/tests/unit/services/test_meal_search_service.py backend/tests/unit/services/test_lodging_meal_selection_service.py backend/tests/contract/test_destination_research_agent_lodging_meals.py backend/tests/integration/test_trip_coordinator_lodging_meals.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实 LLM。Stage 9A 必测用例数为 32，失败数必须为 0。

## 9. 人工验收

不适用：Stage 9A 不接入普通 API/UI；住宿连续性、早餐安排、便餐真实 POI、饮食过滤、餐窗预筛、锚点解析和错误分流均由 FakeAmapProvider、FakeLLM 与固定 fixture 自动化覆盖。显式 shadow 人工入口沿用 Stage 5，不重复建立页面验收。

## 10. 量化退出指标

- [ ] S9A-T01～S9A-T32 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 1～3 日计划均输出恰好 N 个 `LodgingRecord`；Day i `end_hotel_id` 与 Day i+1 `start_hotel_id` 不一致数为 0。
- [ ] 每日早餐安排数量为 1；gourmet 与 lodging_side 并存数为 0；非 gourmet 早餐 POI 数为 0。
- [ ] 已被 gourmet 占用的午晚餐便餐检索次数为 0；未占用午晚餐真实 POI 补齐率为 100%。
- [ ] `LUNCH_SLOT` 必须在午餐选定后解析为真实午餐 POI ID；未解析即检索晚餐的通过数为 0。
- [ ] 所有便餐候选均来自 AmapService 真实餐饮 POI，泛化文本、未知 source ID 或跨日候选通过数为 0。
- [ ] accepted dietary rules 对便餐执行率为 100%；命中规则的便餐入选数为 0；`lodging_side` 早餐被饮食过滤数为 0。
- [ ] 有来源营业时间的 lunch、普通日晚餐、末日晚餐候选均满足对应固定餐窗内连续 `>=40min`；不满足却入选数为 0。
- [ ] 营业时间缺失候选全部标记 `unavailable` 并携带核实提示；默认补营业时间数为 0。
- [ ] 住宿选择和便餐选择一次成功调用数为 1、第三次成功和耗尽均为 3；耗尽不产生 partial 或 fallback。
- [ ] 输出中的完整 ItineraryDraft、路线、最终时刻、费用、地图和 TripPlan 字段数均为 0。
- [ ] Stage 18 前天气变化导致住宿/便餐选择变化的用例数为 0；ignored requirements 进入检索、Prompt 或选择理由数为 0。
- [ ] legacy/shadow 普通请求行为不变，new 仍 reserved；Stage 1～8 回归与前端构建通过。
- [ ] 03 Stage 9A 每条退出条件及 02 的住宿、早餐、便餐、餐窗和饮食规则均映射到测试/证据 ID，新增未追踪 TODO 数为 0。

## 11. 迁移与回滚

- Stage 9A 只在显式新链路内部增加住宿、早餐和便餐选择；默认用户流量仍为 legacy，无持久化数据迁移。
- 回滚时移除 Coordinator 的 Stage 9A 调用、住宿/便餐服务、Agent 入口、错误和新增模型，恢复 Stage 8 PrimaryDraft 作为 bootstrap 末端；Stage 1～8 行为保留。
- 回滚不得删除 Stage 8 PrimaryDraft、Stage 7A/7B 候选与筛选、Stage 5 accepted/ignored constraints 或 dietary rule 表。
- 回滚后 Stage 1～8 全量测试、legacy/shadow 模式测试和前端生产构建必须通过。
- Stage 9B 以后已依赖 `LodgingMealPlan` 时必须按依赖逆序回滚，不能单独移除住宿或便餐输出。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_9a/test-summary.md` | 提交标识、三条命令、32 个用例结果、离线声明和失败数 |
| 住宿连续性矩阵 | `doc/opt_strategy/evidence/stage_9a/lodging-chain-matrix.md` | 1～3 日、A→B、复用酒店、空候选、ID 对账和测试 ID |
| 早餐与餐次矩阵 | `doc/opt_strategy/evidence/stage_9a/breakfast-meal-resolution.md` | gourmet/lodging_side、午晚餐占用、便餐补齐、LUNCH_SLOT 解析 |
| 饮食与餐窗矩阵 | `doc/opt_strategy/evidence/stage_9a/meal-filter-window-matrix.md` | dietary rules、lunch/dinner/末日晚餐餐窗、unavailable 处理 |
| 重试与错误矩阵 | `doc/opt_strategy/evidence/stage_9a/retry-error-matrix.md` | 住宿/便餐选择重试、业务不可行、技术错误和稳定错误码 |
| 边界审计 | `doc/opt_strategy/evidence/stage_9a/stage-boundary-check.md` | Stage 8 输入、9A 输出及未越界进入 9B/路线/时间轴/预算的检查 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_9a/rollback-check.md` | Coordinator 恢复点、Agent 入口撤销、依赖逆序、命令与结果 |

## 13. Ready/Done 复核

当前设计状态：**工作包已细化，待 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 8 实际 Done；审计时重点复核 Stage 8 `PrimaryItineraryDraft` 到本 Stage `LodgingMealPlan` 的 ID/anchor/meal slot 传递、`LUNCH_SLOT` 解析、住宿连续性、饮食与餐窗规则，以及 Stage 9B 对完整活动顺序的唯一职责。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
