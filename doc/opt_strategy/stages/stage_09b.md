# Stage 9B：完整 ItineraryDraft

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 9B |
| 目标 | 第二次调用 ItineraryPlannerAgent，将 PrimaryDraft、连续住宿、早餐安排和便餐选择合并为完整 `ItineraryDraft` 活动顺序 |
| 对应优化项 | P0-4 结构化草案输出、P0-5 餐饮候选检索与筛选、P0-7 行程深度关联化 |
| 前置依赖 | Stage 9A |
| 后续阻断 | 未通过时不得开始 Stage 10A、10B、11、12A、12B、13A、18 |
| 默认流量影响 | 仅扩展显式新链路/bootstrap；legacy 与普通 shadow 请求不生成完整 ItineraryDraft，默认 `/trip/plan` 响应不变 |

## 2. 当前代码基线与差异

- 当前 legacy `ItineraryPlannerAgent` 一次性生成近似完整行程，混合酒店、景点、餐饮和建议，无法证明活动 ID 全部来自本次可信输入。
- Stage 8 已输出 `PrimaryItineraryDraft`，锁定主要点位日期、日内相对顺序、`day_part`、`meal_role`、required、constraint IDs、suggested duration 和 removal rank。
- Stage 9A 已输出 `LodgingMealPlan`，锁定 `lodging[0..N-1]`、每日 `start_hotel_id/end_hotel_id`、早餐安排、缺失午晚餐便餐和餐次解析。
- 当前没有第二次规划调用 schema、完整活动判别联合、住宿/早餐/午晚餐 required 规则、PrimaryDraft 漂移校验或完整草案重试耗尽错误。
- Stage 10A 才计算相邻活动真实路线；Stage 11/12A/12B 才生成时间轴、预算和校验问题；Stage 13A 才组装最终 TripPlan。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 修改 | `backend/app/agents/itinerary_planner_agent.py` | 增加第二次规划 Prompt，只输出完整 ItineraryDraft，不持有工具或生成事实实体 |
| 新增 | `backend/app/services/itinerary_draft_service.py` | 构建 Agent 输入、调用 StructuredLLMAdapter、执行完整草案跨字段校验与有限重试 |
| 新增 | `backend/app/services/itinerary_draft_errors.py` | 定义上游状态、草案校验和重试耗尽等稳定错误码 |
| 修改 | `backend/app/models/itinerary.py` | 增加完整 `ItineraryDraft`、day、activity 判别联合、meal activity 和 lodging boundary 不变量 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 9A 后串行生成 ItineraryDraft 并写入内部规划结果 |
| 新增 | `backend/tests/contract/test_itinerary_planner_full_draft_agent.py` | JSON mode、temperature、schema、Prompt 边界、重试与耗尽测试 |
| 新增 | `backend/tests/unit/services/test_itinerary_draft_service.py` | ID、活动顺序、required、三餐、住宿边界、PrimaryDraft 漂移和非法字段测试 |
| 新增 | `backend/tests/integration/test_trip_coordinator_itinerary_draft.py` | Stage 9A→9B 接入、上游短路、模式回归和下游边界测试 |

本 Stage 不修改 AmapService、Provider、DestinationResearchAgent、Stage 9A 住宿/便餐选择、RouteService、TimelineCalculator、BudgetCalculator、API 路由、legacy Agent、前端类型或页面。

## 4. 输入、输出与错误契约

### 4.1 调用前置与输入快照

- 只允许 Stage 9A 成功输出 `LodgingMealPlan` 后进入本 Stage；Stage 8/9A business_infeasible 或技术错误由 Coordinator 原样短路，Planner 调用数为 0。
- 输入必须包含 request_id、city、start_date、end_date、travel_days、normalized accepted constraints、PrimaryDraft、按 ID 回填的主要点位事实、LodgingMealPlan、酒店事实、早餐安排和便餐事实。
- 输入视图只提供完整排序所需的脱敏事实：稳定 ID、kind/source kind、名称、地址、坐标、required、constraint IDs、suggested duration、PrimaryDraft reason、Stage 9A selection reason 和 data quality。
- ignored requirements 不进入 Prompt、活动理由或约束字段；accepted constraints 只以 normalized 字段和稳定 ID 进入。
- `weather_context` 在 Stage 18 前不得影响完整活动顺序或理由；天气导致重建属于 Stage 18 后重新评审范围。

### 4.2 Agent 调用规范

- 复用 Stage 5 `StructuredLLMAdapter`，使用 JSON mode/`response_format`、`temperature=0.3`；响应字符串直接 `json.loads` 后由 `extra="forbid"` Pydantic 模型校验。
- `ItineraryPlannerAgent.create_itinerary_draft()` 只能引用输入中的 primary candidate ID、lodging ID、breakfast arrangement ID 和 convenience meal ID。
- Agent 可以决定完整活动顺序中 breakfast/gourmet/attraction/convenience meal 的相对位置，但不得改变 PrimaryDraft 中主要点位的日期、相对顺序、`day_part`、`meal_role`、required、constraint IDs 或 removal rank。
- Agent 不得输出地址、坐标、路线、距离、交通耗时、最终时刻、缓冲、价格、预算、地图字段或最终 TripPlan。

### 4.3 输出模型

`ItineraryDraft` 至少包含：

- request_id、city、start_date、end_date、travel_days；
- `days`：长度等于 travel_days，日期严格覆盖 start_date..end_date；
- 每日 `start_hotel_id`、非末日 `end_hotel_id`、末日 `end_hotel_id=null`，必须与 Stage 9A day_boundaries 一致；
- 每日按完整意图排列的 `activities`；
- 派生计数：primary_activity_count、lodging_count、breakfast_count、lunch_count、dinner_count、required_count、optional_count；
- 可选 `draft_notes` 只允许记录排序说明，不得承载事实或可执行承诺。

每个 `ItineraryDraftDay.activities` 使用判别联合：

| activity kind | ID 来源 | required | removal_rank | 规则 |
|---------------|---------|----------|--------------|------|
| `lodging_boundary` | Stage 9A start/end hotel | true | null | 仅表示每日起止酒店边界，不是游览活动；末日无 end boundary |
| `breakfast_lodging_side` | Stage 9A lodging_side breakfast | true | null | 计入早餐完整性，不作为外出 POI 或路线点 |
| `breakfast_gourmet` | PrimaryDraft gourmet breakfast | 与 PrimaryDraft 一致且实际 required=true | null | 作为正式外出活动，必须位于当日所有 attraction 前 |
| `attraction` | PrimaryDraft attraction item | 与 PrimaryDraft 一致 | required 为 null，optional 保留原 rank | 不改变 day_part 和相对顺序 |
| `lunch_gourmet` / `dinner_gourmet` | PrimaryDraft gourmet lunch/dinner | 与 PrimaryDraft 一致且实际 required=true | null | 作为正式餐次活动，同餐次不得再有便餐 |
| `lunch_convenience` / `dinner_convenience` | Stage 9A convenience meal | true | null | 作为正式餐次活动，引用真实餐饮 POI |

### 4.4 住宿与日期边界

- 每日必须有且只有一个 `start_hotel_id`；非末日必须有且只有一个 `end_hotel_id`；末日 `end_hotel_id=null` 且不得生成返店边界。
- Day i 的 `end_hotel_id` 必须等于 Day i+1 的 `start_hotel_id`；同酒店复用和 A→B 换宿均允许。
- `lodging_boundary` 不得携带路线、入住时间、退房时间、费用或房型承诺。
- 完整草案不得新增、删除或替换 Stage 9A 的住宿 ID。

### 4.5 三餐与活动顺序

- 每日必须恰好一个 breakfast、一个 lunch、一个 dinner。
- breakfast 为 `lodging_side` 时不插入外出 POI；breakfast 为 gourmet 时作为正式活动插入当天所有 attraction 前。
- lunch 必须引用 PrimaryDraft 中已占用 lunch 的 gourmet，或 Stage 9A 补齐的 lunch convenience meal；两者不得并存或缺失。
- dinner 必须引用 PrimaryDraft 中已占用 dinner 的 gourmet，或 Stage 9A 补齐的 dinner convenience meal；两者不得并存或缺失。
- `day_part=morning` 的 attraction 必须全部位于 lunch 前，`day_part=afternoon` 的 attraction 必须全部位于 lunch 后；gourmet dinner 必须位于所有 attraction 后。
- 末日晚餐是旅程终点，晚餐后不得输出返店活动或结束酒店。
- 便餐不计为主要点位，不得获得 removal rank；酒店、三餐和必打卡点均不可删除。

### 4.6 PrimaryDraft 漂移校验

- PrimaryDraft 中每个主要点位 item 必须在完整草案中恰好出现一次，日期、kind、required、constraint IDs、suggested duration 和 removal rank 均保持不变。
- 同一日期内 attraction 与 gourmet 的相对约束必须保持：morning、lunch boundary、afternoon、dinner 语义不能被完整草案改写。
- required item 不得获得 removal rank；optional item 的 removal rank 集合必须仍为 PrimaryDraft 的 `1..optional_count`。
- accepted explicit time 目标的 constraint IDs 必须保留；具体时刻可执行性留给 Stage 11/12B。
- 任何漂移均拒绝整份草案并进入有限重试，不允许 Coordinator 局部修补。

### 4.7 确定性输出校验顺序

每次 Agent 响应必须依次通过：

1. JSON、`extra="forbid"` schema 与 request_id/日期对账；
2. days 数量、连续日期和 day boundary 与 Stage 9A 对账；
3. PrimaryDraft item 全集、日期、kind、required、constraint IDs、duration、removal rank 对账；
4. 每日 breakfast/lunch/dinner 三餐唯一性和来源对账；
5. lodging_side breakfast 非 POI、gourmet breakfast 外出活动和午晚餐正式活动规则；
6. day_part、meal_role、同餐次互斥和末日晚餐旅程终点规则；
7. required 与 optional 删除排名规则；
8. 输出不含地址、坐标、路线、时刻、费用、预算、地图或 TripPlan 字段；
9. ignored 与天气未进入决策字段或理由。

任何一项失败均拒绝整份 ItineraryDraft 并进入有限重试；不得返回部分日期或 partial draft。

### 4.8 有限重试、错误与日志

- 首次失败后最多重试 2 次，总调用上限 3；schema/JSON、日期、ID、住宿边界、三餐、PrimaryDraft 漂移、非法字段或计数错误均可重试。
- 重试使用相同输入快照、temperature 和 schema，只追加稳定脱敏校验码；不得回显完整 Prompt、自由文本、POI事实、酒店事实或上次完整输出。
- 任一合法响应立即停止。三次均失败时抛 `ItineraryDraftError(error_code="itinerary_draft_retry_exhausted")`，属于核心规划技术错误；不得转 ignored、business_infeasible、partial 或 fallback。
- 日志按 request_id 记录 stage=`itinerary_draft`、attempt、day/activity/meal/required count、validation/error code、status 和 duration；不得记录 Key、Prompt、自由文本、完整候选、完整 Agent 输出或完整事实 payload。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或功能开关；temperature=0.3、最多重试 2 次、三餐唯一性、PrimaryDraft 漂移拒绝和非法字段拒绝是受测试锁定的领域策略。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求不运行 Coordinator；显式 shadow 验证可执行 Stage 9B；new 仍为 reserved。
- 复用现有 LLM 配置和 StructuredLLMAdapter 注入；ItineraryPlannerAgent 不读取 Settings、不持有外部工具。

## 6. 实施步骤

1. 定义完整 `ItineraryDraft`、day、activity 判别联合、计数派生和稳定错误码，补序列化正反例。
2. 建立 ItineraryPlannerAgent 第二次调用 Prompt，固定 temperature=0.3、JSON schema、输入 ID 白名单和禁止输出字段。
3. 实现输入快照与 deterministic preflight，覆盖 Stage 9A 状态、day boundaries、meal_slot_resolution、accepted constraints 和天气零影响。
4. 实现住宿边界、三餐唯一性、`lodging_side` 早餐和午晚餐来源对账。
5. 实现 PrimaryDraft 漂移校验：ID、日期、kind、required、constraint IDs、duration、removal rank 与相对顺序。
6. 实现 day_part/meal_role 活动顺序校验、末日晚餐终点和禁止返店规则。
7. 实现最多 2 次重试、稳定脱敏错误反馈和 `itinerary_draft_retry_exhausted` 技术错误。
8. 接入 TripCoordinator 并补 Stage 9A→9B 集成回归；执行 Stage 1～9B 测试与前端构建，归档证据。

## 7. 明确不做

- 不重新选择、添加、删除或替换主要点位、住宿、早餐或便餐；如需改变必须回到对应上游 Stage 重建。
- 不调用 AmapService、DestinationResearchAgent、RouteService、TimelineCalculator 或 BudgetCalculator。
- 不生成真实路线、距离、交通耗时、最终时刻、缓冲、预算、地图 polyline 或最终 TripPlan。
- 不校验午晚餐最终实际时间窗或美食级早餐营业时间覆盖；属于 Stage 11/12B。
- 不处理 partial/failed 结果状态；属于 Stage 13D/16。
- 不让天气影响完整活动顺序；Stage 18 接入时必须按天气重建边界重新评审。
- 不修改普通 API、前端、legacy 链路，不开放 new 模式，不删除 legacy fallback。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S9B-T01 | contract | 合法 1～3 日 ItineraryDraft | JSON mode、temperature=0.3、extra=forbid，日期完整 |
| S9B-T02 | contract | 输出地址/坐标/路线/时刻/费用/TripPlan | schema 拒绝并重试 |
| S9B-T03 | unit | 上游 9A business_infeasible | Planner 调用数 0，状态原样短路 |
| S9B-T04 | unit | day boundaries 连续 | Day i end_hotel 等于 Day i+1 start_hotel |
| S9B-T05 | unit | 末日 end_hotel 非空或返店活动 | 拒绝 |
| S9B-T06 | unit | 每日三餐完整 | breakfast/lunch/dinner 各 1 个 |
| S9B-T07 | unit | lodging_side 早餐 | 计入早餐但不是 POI/路线活动 |
| S9B-T08 | unit | gourmet 早餐 | 位于所有 attraction 前，作为正式活动 |
| S9B-T09 | unit | lunch gourmet vs convenience | 二者恰好一个，重复/缺失拒绝 |
| S9B-T10 | unit | dinner gourmet vs convenience | 二者恰好一个，重复/缺失拒绝 |
| S9B-T11 | unit | 末日晚餐终点 | 晚餐后无返店和后续活动 |
| S9B-T12 | unit | PrimaryDraft ID 遗漏/重复/未知 | 均拒绝 |
| S9B-T13 | unit | PrimaryDraft 日期或 kind 漂移 | 拒绝 |
| S9B-T14 | unit | required/constraint IDs 漂移 | 拒绝 |
| S9B-T15 | unit | suggested duration 漂移 | 拒绝 |
| S9B-T16 | unit | required removal rank 非空 | 拒绝 |
| S9B-T17 | unit | optional removal rank 漂移 | 拒绝 |
| S9B-T18 | unit | morning/afternoon 与 lunch 顺序 | morning 全在 lunch 前，afternoon 全在 lunch 后 |
| S9B-T19 | unit | gourmet dinner 位置 | 位于所有 attraction 后 |
| S9B-T20 | unit | 便餐获得 removal rank | 拒绝 |
| S9B-T21 | unit | 酒店、三餐、必打卡 required | 全部 required 且不可删除 |
| S9B-T22 | contract | 首次成功、第三次成功 | 调用数分别为 1、3，输入快照不变 |
| S9B-T23 | contract | 三次校验失败 | itinerary_draft_retry_exhausted，不返回 partial draft |
| S9B-T24 | contract | LLM 技术异常耗尽 | 保持技术错误，不转 business_infeasible/ignored/fallback |
| S9B-T25 | integration | Stage 9A→9B happy path | Coordinator 保存 ItineraryDraft，供 Stage 10A 使用 |
| S9B-T26 | integration | ignored 和天气变化 | Stage 18 前完整草案输入与输出一致 |
| S9B-T27 | regression | legacy/shadow/new 模式 | 普通 legacy/shadow Planner 调用数 0，new 仍 planner_mode_not_ready |
| S9B-T28 | regression | Stage 1～9A 与前端 | 上游测试和生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_itinerary_draft_service.py backend/tests/contract/test_itinerary_planner_full_draft_agent.py backend/tests/integration/test_trip_coordinator_itinerary_draft.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实 LLM。Stage 9B 必测用例数为 28，失败数必须为 0。

## 9. 人工验收

不适用：Stage 9B 不接入普通 API/UI；住宿边界、三餐完整性、活动顺序、PrimaryDraft 漂移、非法字段和重试均由 FakeLLM 与固定 fixture 自动化覆盖。显式 shadow 人工入口沿用 Stage 5，不重复建立页面验收。

## 10. 量化退出指标

- [ ] S9B-T01～S9B-T28 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] ItineraryDraft days 精确覆盖请求 1～3 日，缺日、重复日、越界日通过数为 0。
- [ ] Day i `end_hotel_id` 与 Day i+1 `start_hotel_id` 不一致数为 0；末日结束酒店和返店活动数为 0。
- [ ] 每日 breakfast/lunch/dinner 数量均为 1；重复餐次和缺失餐次通过数为 0。
- [ ] PrimaryDraft item 安排率为 100%，每个 ID 恰好出现一次；未知、遗漏、重复或漂移通过数为 0。
- [ ] required、constraint IDs、suggested duration 和 removal rank 与 PrimaryDraft 一致率为 100%。
- [ ] 酒店、三餐、必打卡点 required 且 removal_rank=null；便餐 removal rank 非空通过数为 0。
- [ ] `lodging_side` 早餐 POI/路线活动数为 0；gourmet 早餐外出活动位置正确率 100%。
- [ ] 输出中的地址、坐标、路线、最终时刻、费用、预算、地图和 TripPlan 字段数均为 0。
- [ ] 一次成功调用数为 1、第三次成功和耗尽均为 3；耗尽全部返回 itinerary_draft_retry_exhausted 技术错误。
- [ ] Stage 18 前天气变化导致完整草案变化的用例数为 0；ignored requirements 进入 Prompt/草案数为 0。
- [ ] legacy/shadow 普通请求行为不变，new 仍 reserved；Stage 1～9A 回归与前端构建通过。
- [ ] 03 Stage 9B 每条退出条件及 02 的完整草案、住宿、三餐、required 和数据责任规则均映射到测试/证据 ID，新增未追踪 TODO 数为 0。

## 11. 迁移与回滚

- Stage 9B 只在显式新链路内部增加完整 ItineraryDraft；默认用户流量仍为 legacy，无持久化数据迁移。
- 回滚时移除 Coordinator 的 ItineraryDraft 调用、ItineraryPlannerAgent 第二次调用入口、service/error 和新增模型细化，恢复 Stage 9A `LodgingMealPlan` 作为 bootstrap 末端；Stage 1～9A 行为保留。
- 回滚不得删除 PrimaryDraft、LodgingMealPlan、住宿、早餐、便餐、accepted constraints 或 dietary rule 表。
- 回滚后 Stage 1～9A 全量测试、legacy/shadow 模式测试和前端生产构建必须通过。
- Stage 10A 以后已依赖 ItineraryDraft 时必须按依赖逆序回滚，不能单独移除完整草案或改变活动 ID 语义。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_9b/test-summary.md` | 提交标识、三条命令、28 个用例结果、离线声明和失败数 |
| 完整草案契约矩阵 | `doc/opt_strategy/evidence/stage_9b/itinerary-draft-contract.md` | day、activity、ID、required、rank、非法字段与测试 ID |
| 住宿与三餐矩阵 | `doc/opt_strategy/evidence/stage_9b/lodging-meal-order-matrix.md` | 起止酒店、早餐、午晚餐、末日边界和活动顺序 |
| PrimaryDraft 漂移矩阵 | `doc/opt_strategy/evidence/stage_9b/primary-draft-drift-matrix.md` | ID、日期、kind、constraint、duration、rank 和顺序对账 |
| 重试与错误矩阵 | `doc/opt_strategy/evidence/stage_9b/retry-error-matrix.md` | 1/3/耗尽、技术错误、上游短路和稳定错误码 |
| 边界审计 | `doc/opt_strategy/evidence/stage_9b/stage-boundary-check.md` | 9A 输入、9B 输出及未越界进入路线/时间轴/预算/TripPlan 的检查 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_9b/rollback-check.md` | Coordinator 恢复点、Agent 入口撤销、依赖逆序、命令与结果 |

## 13. Ready/Done 复核

当前设计状态：**工作包已细化，待 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 9A 实际 Done；审计时重点复核 Stage 8 `PrimaryItineraryDraft`、Stage 9A `LodgingMealPlan` 到本 Stage `ItineraryDraft` 的 ID、required、三餐、住宿边界和活动顺序传递，以及 Stage 10A 对路线计算的唯一职责。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
