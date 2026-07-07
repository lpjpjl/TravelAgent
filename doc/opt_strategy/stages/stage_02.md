# Stage 2：核心领域契约
> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 2 |
| 目标 | 用严格 Pydantic 契约固定请求、来源、约束、候选、草案、路线、时间轴、预算、计划状态和错误边界，为 Stage 3 以后提供唯一模型语义 |
| 对应优化项 | P0-4、P1-1、P1-2 的契约前置，并承接 P0-1、P0-5～P0-9 的模型基础 |
| 前置依赖 | Stage 1 的 Provider 原始结果与错误边界完成验证；Stage 2 可先写规格，但代码开工前 Stage 1 必须通过 Gate 依赖要求 |
| 后续阻断 | 未通过时不得开始 Stage 3～16、18、20、23、28、31A～31C 的契约实现 |
| 默认流量影响 | 无；只新增目标领域模型和离线契约测试，不切换 `/trip/plan`，不修改当前前端类型或页面 |

## 2. 当前代码基线与差异

- 当前所有后端模型集中在 `backend/app/models/schemas.py`，请求、外部事实、展示对象和 API 包装没有分层；多数模型沿用宽松字符串和默认空值。
- `TripRequest` 允许 1～30 天，日期是未做跨字段校验的字符串，交通、住宿和偏好均为任意字符串，偏好数量没有上限；目标约束为 1～3 天、最多 3 个偏好和前后端一致的受控枚举。
- `Location` 未声明 GCJ-02、经纬度范围或顺序语义；`POIInfo` 缺少来源通道、采集时间、数据质量、原始类型和营业时间。
- `Attraction.poi_id`、酒店地址/评分、费用等字段使用 `""` 或 `0` 伪装缺失；`WeatherInfo` 将无法解析温度回退为 `0`，无法区分真实零度与不可用事实。
- `Meal`、`Hotel`、`DayPlan` 与 `TripPlan` 不能表达来源、required、约束关联、连续住宿、路线、时间轴、partial/failed、degraded 和缺失事实；预算仅有整数汇总，缺少人数、房间、晚数、单价、数量、计价单位与依据。
- 当前 `ErrorResponse` 只有可选错误码，没有 `request_id`、`retryable`、结构化 details 和 HTTP/业务状态分流。
- `frontend/src/types/index.ts` 手工复制旧模型，交通和住宿仍为普通字符串。Stage 2 只固化后端权威契约及枚举清单，不提前迁移前端；Stage 13E/16 再按该契约接入当前行程状态和新 API schema。
- 当前没有 ExtraConstraint、候选/草案、路线、时间、预算明细、来源质量或 schema 正反例测试。

## 3. 文件变更清单

Stage 2 按领域职责拆分模型，避免继续扩张单一 `schemas.py`。文件名在本工作包冻结后视为目标路径；后续 Stage 只增量扩展，不另建同义模型。

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/models/common.py` | 严格基类、ID、日期时间、坐标、金额、来源元数据、质量/新鲜度与公共枚举 |
| 新增 | `backend/app/models/request.py` | `TripRequest`、交通/住宿/偏好受控枚举及 1～3 天跨字段校验 |
| 新增 | `backend/app/models/constraint.py` | 四类 `ExtraConstraint`、accepted/ignored 解析结果、稳定原因码与修订冲突模型 |
| 新增 | `backend/app/models/source.py` | POI、营业时间、天气和外部事实来源模型；只定义领域契约，不实现 Provider payload 解析 |
| 新增 | `backend/app/models/candidate.py` | 结构化候选、候选类型、命中偏好、required、约束关联与空间锚点 |
| 新增 | `backend/app/models/itinerary.py` | Primary/完整草案、连续住宿、两类早餐、餐次、活动和删除排名契约 |
| 新增 | `backend/app/models/route.py` | 请求偏好、实际路段方式、route ID、polyline、距离、耗时和策略元数据 |
| 新增 | `backend/app/models/timeline.py` | 带 `+08:00` 的活动/路线时间片、日时间轴和问题引用结构 |
| 新增 | `backend/app/models/budget.py` | Decimal 金额、费用项、计价单位、固定人数/房间假设和已知费用汇总 |
| 新增 | `backend/app/models/trip.py` | `PlanStatus`、missing items、degraded、normalized accepted constraints 和最终 `TripPlan` |
| 新增 | `backend/app/models/error.py` | 统一 API 错误响应、稳定错误码及 retryable/details 边界 |
| 修改 | `backend/app/models/__init__.py` | 只导出跨层稳定公共类型，避免业务模块继续从旧聚合文件随意取模型 |
| 保留 | `backend/app/models/schemas.py` | Stage 16 前继续服务旧链路；本 Stage 不删除、不改名、不让新模型反向依赖旧模型 |
| 新增 | `backend/tests/unit/models/test_common.py` | request_id、枚举、日期时间、坐标、金额、quality/freshness 测试 |
| 新增 | `backend/tests/unit/models/test_request.py` | TripRequest 与跨字段输入校验测试 |
| 新增 | `backend/tests/unit/models/test_constraint.py` | 四类约束、accepted/ignored 与冲突模型测试 |
| 新增 | `backend/tests/unit/models/test_source_candidate.py` | 来源事实、缺失原因、候选和空间锚点测试 |
| 新增 | `backend/tests/unit/models/test_itinerary.py` | 两次草案、餐次、早餐和连续住宿测试 |
| 新增 | `backend/tests/unit/models/test_route_timeline.py` | 路线、polyline、时间片与时区测试 |
| 新增 | `backend/tests/unit/models/test_budget_trip_error.py` | Decimal 预算、计划状态、missing items 和错误响应测试 |
| 新增 | `backend/tests/contract/test_domain_serialization.py` | 固定关键 JSON 形状、枚举值、null 语义和跨模型引用 |

本 Stage 不修改 `frontend/src/types/index.ts`。前后端一致性通过本工作包记录的稳定枚举值和 JSON 示例约束，实际 TypeScript 接入由 Stage 13E/16 完成。

## 4. 输入、输出与错误契约

### 4.1 全局建模规则

- 新模型统一使用 Pydantic v2 严格配置：`extra="forbid"`；会参与 API/Agent JSON 的枚举序列化为稳定英文值，不接受中文别名或未知值静默透传。
- 字符串必填事实去除首尾空白后不得为空；可缺失事实使用 `Optional[...] = None`，禁止以 `0`、`""`、空坐标或占位名称表达 unavailable。
- 日期使用 `date`，时间点使用带时区 `datetime`；目的地时区固定 `Asia/Shanghai`，序列化必须带 `+08:00`。禁止 naive datetime，禁止仅靠 `day_index` 判断末日。
- 金额统一使用 `Decimal`、币种 `CNY`、两位小数；JSON 使用十进制定点字符串，避免二进制浮点和前后端精度漂移。
- 所有跨模型引用使用稳定 ID；事实对象保留 `source_id`，决策对象引用候选/事实 ID，不复制后再允许字段分叉。
- `DataQuality` 严格采用 02 的五类语义：`sourced/computed/estimated/generated/unavailable`；`Freshness` 独立表达 `live/cached_fresh/cached_stale`。Provider 只是技术通道，不能把 MCP/HTTP 名称当作真实性等级。
- `request_id` 使用服务端生成的 UUID4 字符串，在 API、Coordinator、Provider、Agent、日志和错误间原样传播；它不是用户字段，不进入事实 ID、constraint ID、route ID 或缓存键。
- `degraded=true` 仅由 stale 缓存或设计允许的降级数据路径触发，与 `plan_status` 正交；人工切换 Provider 不触发 degraded。

### 4.2 请求与公共枚举

| 契约 | 稳定值/规则 |
|------|-------------|
| `TripRequest` | 城市非空；`travel_days` 为 1～3；`travel_days == (end_date - start_date).days + 1`；`start_date <= end_date`；偏好最多 3 个且不允许重复；`free_text_input` 去除首尾空白后最多 1000 个 Unicode code point |
| `TransportationPreference` | `walking / public_transit / self_drive / mixed`；表示默认偏好，不是系统硬约束 |
| `AccommodationPreference` | `economy_hotel / comfort_hotel / luxury_hotel / homestay`，依次映射当前“经济型酒店/舒适型酒店/豪华酒店/民宿”；未列值拒绝 |
| `TravelPreference` | `history_culture / natural_scenery / food / shopping / art / leisure`，依次映射当前六个首页标签；最多 3 个且保持用户选择顺序 |
| `AmapProviderName` | `mcp / http`；仅描述来源技术通道 |
| `CoordinateSystem` | 领域输出固定 `gcj02`；Provider 原始坐标系必须显式记录，非 GCJ-02 只能在 Provider 层转换 |
| `PlanStatus` | `complete / partial / failed` |
| `DataQuality` | `sourced / computed / estimated / generated / unavailable` |
| `Freshness` | `live / cached_fresh / cached_stale` |

`failed` 是可信业务结果而非 API 技术异常；请求字段非法仍返回 422。`partial` 只能表达从 Day 1 开始的最长连续完整日前缀，不能绕过 accepted 约束。

### 4.3 来源事实与候选

- `SourceMetadata` 至少包含 provider、operation、抓取时间、freshness 和 data quality；POI 等具有上游实体 ID 的事实必须包含 source ID，天气按预报日期标识，路线使用原生或 Stage 4 生成的 route ID。原始类型和坐标系只在适用对象上必填；可选事实若 unavailable，必须有稳定缺失原因，不得伪造值。
- `GeoPoint` 字段顺序固定为 `longitude/latitude`，分别校验合法范围，领域坐标系只能是 GCJ-02；route polyline 使用同一坐标对象列表，不接受未声明坐标系的裸数组。
- `SourcedPOI` 必须包含真实 source ID、具体名称、地址、合法坐标和 POI 类型；餐饮分店名称不得退化为检索词或类别名。
- 营业时间使用按星期关联的一个或多个区间表达，必须区分跨午夜、`24:00` 和不可解析/缺失；原始值不可可靠解析时为 unavailable，不交给 LLM 推断。
- `Candidate` 引用真实事实 ID，记录候选类型、命中偏好、是否 required、constraint IDs 和筛选元数据。候选层不得复制并改写 POI 名称、地址或坐标。
- 空间锚点封闭为 `POI_ID / DAY_START / LUNCH_SLOT / DAY_END`；只有 `POI_ID` 携带 POI 引用，其余锚点由后续草案上下文解析。

### 4.4 额外要求契约

- `ExtraConstraint` 是以 `type` 为判别字段的封闭联合，仅允许：`must_visit_poi`、`dietary_exclusion`、`explicit_time_constraint`、`mobility_constraint`。
- 每个 accepted 约束必须有稳定 `constraint_id`、规范化执行字段和 trip scope；原始完整自由文本不得复制进 TripPlan、日志或验收 fixture。
- `ExtraConstraintParseResult` 按子句输出 accepted 或 ignored。ignored 必须包含稳定原因码，至少覆盖 `unsupported`、`ambiguous`、`incomplete`、`parse_failed`；ignored 不得进入候选、草案或最终 accepted constraints。
- `must_visit_poi` 必须最终关联唯一真实 POI ID并标记 required；无法唯一匹配或最终未安排时整单 failed。
- accepted `explicit_time_constraint` 必须关联 required 目标和带 `+08:00` 的时间窗口；字段不全时只能 ignored，不能凭猜测补齐。
- `dietary_exclusion` 只保存封闭规则 ID，由确定性规则表执行；校验范围不包含 `lodging_side` 早餐。
- `mobility_constraint` 只允许可用真实路线距离验证的限制；不支持的无障碍语义进入 ignored。系统默认 `MAX_WALKING_SEGMENT_METERS=2000` 属于系统硬约束，不伪装成用户约束。
- 约束优先级固定为：系统硬约束 → accepted 额外要求 → 三级选址优先级/普通偏好 → Planner 建议。低优先级对象不得覆盖高优先级约束。

### 4.5 草案、住宿与餐次

- `PrimaryItineraryDraft` 只包含主要点：非餐饮点必须有 `day_part=morning|afternoon`，美食级餐点必须有 `meal_role=breakfast|lunch|dinner`；每天至少一个景点或美食级主要点。
- `provisional_meal_slots` 明确三餐中哪些已被美食级餐点占用、哪些待便餐/住宿侧早餐补齐；同一餐次不得重复安排。
- 完整 `ItineraryDraft` 才加入起止酒店、连续住宿和未占用餐次。酒店、三餐、必打卡点为 required；只有其他主要点允许 `removal_rank`，且排名在同一修复上下文中唯一。
- `lodging` 长度即 `lodging_count`：complete N 日为 N；partial Day1..DayK 且 K<N 时为 K+1。相邻日的 `end_hotel_id` 必须等于次日 `start_hotel_id`。
- `BreakfastArrangement` 为判别联合：`lodging_side` 不得包含 POI ID、地址、坐标或路线；`gourmet` 必须引用真实餐饮 POI并进入路线、营业时间、时间轴和预算。
- 普通景点建议时长为 30～240 分钟，在外餐饮为 40～120 分钟；缺失或越界属于非法结构化输出，Calculator 不补默认值。

### 4.6 路线、时间轴与预算

- 请求交通偏好与实际路段方式分离。实际路段方式为 `walking/taxi/transit/self_drive`；`transit` 仅按来源事实细分地铁、公交或旅游专线，`mixed` 不能生成 self-drive 路段。
- walking 路段距离超过 2000m 时只淘汰该方式并触发 taxi/transit 候选，不直接让日期失败。回退稳定排序键为 `(duration_seconds, distance_meters, route_id)`。
- `RouteSegment` 必须包含起终点引用、实际方式、距离、耗时、polyline、来源/策略元数据和 route ID；无原生 ID 时的确定性生成规则由 Stage 4 实现，但 schema 必须区分 source/generated。
- 时间轴由活动片和真实路线片组成，所有时间带 `+08:00`；午餐、普通日晚餐、末日晚餐的窗口及非餐饮点不得跨午餐等规则由 TimelineValidator 执行，模型不得用自由文本替代时间字段。
- `BudgetAssumption` 固定 2 名成人、1 间房及实际 lodging_count；费用项包含类别、单价、数量、计价单位、小计、币种、来源依据和 quality。
- unavailable 费用不进入 known total；预算必须命名为“已知费用合计”并携带缺失项，不能把它描述为完整总预算。公共交通按人、打车按车、住宿按房晚表达不同计价单位。

### 4.7 最终计划、partial 与错误

- `TripPlan` 必须携带 plan status、degraded、完整 normalized accepted constraints、日期范围、返回日、连续 lodging、路线、时间轴、预算、missing items 和来源提示。
- `complete` 包含请求的全部 N 个完整日；`partial` 只允许 Day1..DayK 的最长连续完整前缀且 K<N；首日不可行、accepted 约束不满足或美食偏好数量硬规则不满足时为 failed。
- partial 中每个返回日仍必须具备酒店、三餐、必要路线和完整时间轴；可选评分、票价、营业时间等 unavailable 不影响完整日判定，但必须进入 missing items/核实提示。
- 美食偏好开启时，complete 含 1～N 个美食级餐点，partial 含 1～K 个且每天最多一个；该约束必须能由 schema 和跨字段校验表达。
- `ApiErrorResponse` 固定包含 `request_id/error_code/message/retryable/details`。字段错误映射 422、修订冲突 409、上游非法响应 502、配置/服务不可用 503、超时 504；details 只能是稳定、脱敏结构。
- 合法空候选或真实安排不可行进入业务 `partial/failed`；技术链路在重试和缓存降级后仍无可信输入才返回 API 错误。额外要求解析失败是唯一明确例外，以 `parse_failed` ignored 继续。
- `RevisionConstraintConflict` 使用稳定错误码 `revision_constraint_conflict`，冲突响应不得携带新版 TripPlan。

## 5. 配置与功能开关

- 本 Stage 不新增 Settings 字段、环境变量或功能开关；领域常量不得通过环境变量改变业务语义。
- `DESTINATION_TIMEZONE="Asia/Shanghai"`、`CURRENCY="CNY"`、`MAX_WALKING_SEGMENT_METERS=2000`、成人数 2 和房间数 1 是已冻结的领域常量，定义在目标模型/常量模块并由测试锁定。
- `AMAP_PROVIDER` 与 `AMAP_API_KEY` 沿用 Stage 1；Stage 2 模型只接受来源元数据，不读取配置，也不创建 Provider。
- 旧 `/trip/plan` 继续使用 `schemas.py`，无需为纯新增模型设置流量开关。Stage 16 才负责默认链路和前端 schema 同步切换。

## 6. 实施步骤

1. 在 `common.py` 建立严格模型基类、稳定枚举、时区日期时间、GCJ-02 坐标、Decimal 金额、来源质量与新鲜度类型，先完成公共正反例测试。
2. 在 `request.py` 固化首页枚举映射、1～3 天、日期一致性和最多 3 个偏好规则；保留用户偏好顺序，重复值直接校验失败，不静默去重。
3. 在 `constraint.py` 建立四类封闭约束、accepted/ignored 判别结果、原因码和冲突响应；仅定义结构，不实现 LLM 解析或饮食规则表。
4. 在 `source.py` 与 `candidate.py` 定义真实事实、营业时间、候选、required/constraint 引用和空间锚点；不解析 Stage 1 payload。
5. 在 `itinerary.py`、`route.py`、`timeline.py` 和 `budget.py` 定义两次草案、连续住宿、早餐联合、实际路线、时间片和预算明细，并加入能在模型层确定判断的跨字段校验。
6. 在 `trip.py` 与 `error.py` 定义 complete/partial/failed、degraded、missing items、accepted constraints、最终计划和统一错误；业务算法才能判断的条件保留给 Coordinator/Validator，不在 schema 中猜测执行结果。
7. 固化关键 JSON 正反例和序列化契约，验证新模型不导入旧 `schemas.py`，旧链路仍可独立导入；最后执行 Stage 1 回归与前端构建。

## 7. 明确不做

- 不接入 MCP/HTTP Provider，不解析任何真实高德 payload，不消除 `AmapService` TODO。
- 不实现额外要求 LLM 解析、候选检索/筛选、Planner、Coordinator、路线选择、时间/预算计算或修复算法。
- 不修改 `frontend/src/types/index.ts`、页面、API 请求或 `/trip/plan` 响应，不让旧链路返回新模型。
- 不删除、重命名或批量迁移 `backend/app/models/schemas.py`；不为兼容旧模型放宽新模型的严格校验。
- 不把只有运行算法才能确认的“可执行”“最优”“最长 partial”伪装成单模型字段校验；本 Stage 只验证其结构可表达性和可由后续服务消费的引用完整性。
- 不引入数据库、schema 生成器、OpenAPI 客户端生成或前端测试框架。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S2-T01 | unit | TripRequest 1、2、3 天合法样本 | 日期差、travel_days、枚举和偏好通过，序列化稳定 |
| S2-T02 | unit | 0/4 天、日期倒置、天数不一致、额外要求超过 1000 code point | 全部拒绝并定位到稳定字段/跨字段错误 |
| S2-T03 | unit | 六类偏好、四类住宿、四类交通及未知值 | 已列英文枚举通过；中文值、未知值拒绝；偏好超过 3 个或重复拒绝 |
| S2-T04 | unit | GeoPoint 边界、经纬颠倒与非 GCJ-02 领域坐标 | 合法边界通过；非法范围、顺序异常样本和非 GCJ-02 拒绝 |
| S2-T05 | unit | 五类 DataQuality 与 freshness/degraded 组合 | 五类语义互斥且稳定；缺失事实为 null 且有原因；stale 可独立触发 degraded，不改变 plan status |
| S2-T06 | unit | 四类 ExtraConstraint accepted/ignored | 判别联合严格；accepted 有稳定 ID/执行字段；ignored 原因码完整且不能混入 accepted |
| S2-T07 | unit | 必打卡、时间、饮食和移动约束非法形状 | 缺少 required 字段、额外字段、naive 时间或不支持规则 ID 均拒绝 |
| S2-T08 | unit | PrimaryDraft 与完整 ItineraryDraft | day_part/meal_role、餐次占用、required、removal_rank 和起止酒店引用满足契约 |
| S2-T09 | unit | 早餐判别联合 | lodging_side 携带 POI/路线时拒绝；gourmet 缺少真实 POI 引用时拒绝 |
| S2-T10 | unit | 连续住宿 complete N 与 partial K+1 | 合法长度和相邻酒店连续性通过；断链、晚数不符拒绝 |
| S2-T11 | unit | 活动时长上下界 | 景点 30/240、餐饮 40/120 通过；缺失或越界拒绝 |
| S2-T12 | unit | RouteSegment 与时间片 | 距离/耗时非负、polyline 为 GCJ-02、实际方式和 source/generated ID 形状正确；naive datetime 拒绝 |
| S2-T13 | unit | Decimal 预算与 unavailable 费用 | 2 成人/1 房、实际晚数、计价单位和已知合计一致；浮点输入、错误小计和 unavailable 计入合计拒绝 |
| S2-T14 | unit | complete/partial/failed 及 missing items | 三种状态结构可表达；partial 必须从 Day1 连续、failed 不伪装完整 days |
| S2-T15 | contract | ApiErrorResponse 状态样本 | 422/409/502/503/504 JSON 形状稳定、details 脱敏，冲突响应无新版 TripPlan |
| S2-T16 | contract | 关键 JSON 往返与额外字段 | Decimal/枚举/datetime/null 序列化稳定；反序列化等价；任意额外字段拒绝 |
| S2-T17 | contract | 新旧模型隔离 | 新模型不导入旧 `schemas.py`，旧模型仍可独立导入，新契约不会改变旧入口行为 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/models backend/tests/contract/test_domain_serialization.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、读取真实 Key 或调用真实 LLM。Stage 2 必测用例数为 17，失败数必须为 0。

## 9. 人工验收

不适用：Stage 2 只新增离线领域模型，不接入 API 或 UI；确定性输入、输出、错误和序列化行为全部由 S2-T01～S2-T17 覆盖。评审者只需在证据摘要中核对模型 JSON 示例与 02 的业务规则映射，不另设主观页面验收。

## 10. 量化退出指标

- [ ] S2-T01～S2-T17 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 请求只接受 1～3 天、四类交通、四类住宿、六类偏好且偏好最多 3 个；未知值、中文传输值和重复偏好均被拒绝。
- [ ] 四类 ExtraConstraint、accepted/ignored、稳定原因码、required 和优先级引用均可由严格 schema 表达。
- [ ] sourced/computed/estimated/generated/unavailable、freshness、null、GCJ-02 和带 `+08:00` 时间不存在互相冒充或以空字符串、0、naive datetime、裸坐标伪装的路径。
- [ ] PrimaryDraft、完整 ItineraryDraft、连续住宿、两类早餐、真实路线、时间轴和预算明细均有至少一个合法与一个非法样本。
- [ ] complete N、partial Day1..DayK + K+1 住宿、failed、degraded 和 missing items 的组合可表达且不会互相冒充。
- [ ] 预算只使用 Decimal/CNY，2 名成人、1 间房、实际 lodging_count 和不同交通计价单位均进入契约；unavailable 不计入已知费用合计。
- [ ] 统一错误响应覆盖 422/409/502/503/504；业务不可行仍由 200 + failed 表达；冲突响应不包含新版 TripPlan。
- [ ] 新模型对旧 `schemas.py` 单向隔离，旧 `/trip/plan` 和当前前端构建不受影响；新增未追踪 TODO 数为 0。
- [ ] 03 Stage 2 范围及 02 第 4.1～4.7 节的真实性、来源、状态组合均映射到具体模型和测试 ID。

## 11. 迁移与回滚

- 本 Stage 为旁路新增模型，默认入口不引用它们；回滚删除新增领域模型及对应测试，并恢复 `models/__init__.py` 导出即可，不涉及 API、前端状态、缓存或持久化数据。
- `schemas.py` 在 Stage 16 前保持旧链路权威模型，禁止本 Stage 做双向继承或自动转换，因此回滚不会要求数据迁移。
- 回滚后必须验证旧后端模型可导入、Stage 1 测试仍通过、前端生产构建成功；若 Stage 1 测试尚未实际建立，则使用工作包中已定义的最小导入检查并在实施时将精确命令写入证据。
- 任何后续 Stage 已开始依赖新模型后，不得单独回滚 Stage 2；必须按依赖逆序回滚到最后一个不引用新模型的提交。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_2/test-summary.md` | 提交标识、三条命令、17 个用例结果、离线声明和失败数 |
| 契约映射 | `doc/opt_strategy/evidence/stage_2/schema-matrix.md` | 02 第 4.1～4.7 节 → 模型 → 字段/校验 → 测试 ID |
| JSON 示例 | `doc/opt_strategy/evidence/stage_2/serialization-summary.md` | 脱敏最小请求、complete/partial/failed、错误响应形状及枚举表，不复制大 fixture |
| 回滚验证 | `doc/opt_strategy/evidence/stage_2/rollback-check.md` | 回滚边界、依赖逆序要求、命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计复核结论：**R2～R10 通过首轮跨 Stage 审计；执行 Ready 待 R1**。R1 必须等待 Stage 1 实际 Done，并在 Stage 2 开工前再次确认 R2～R10 未因 Stage 1 实现结论变化而失效；当前不表示 Ready、Done、设计冻结或允许修改工程代码。
