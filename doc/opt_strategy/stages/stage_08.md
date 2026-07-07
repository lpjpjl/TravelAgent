# Stage 8：PrimaryItineraryDraft

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 8 |
| 目标 | 让 ItineraryPlannerAgent 将 Stage 7B 筛选后的全部主要点位按请求日期分配并排序，生成可严格校验的第一次草案及后续酒店、便餐检索所需锚点 |
| 对应优化项 | P0-4 结构化草案输出、P0-7 行程深度关联化 |
| 前置依赖 | Stage 2、5、7B |
| 后续阻断 | 未通过且未完成 Stage 6～8 跨 Stage 审计时，不得开始 Stage 9A、9B、10A、13A、18 |
| 默认流量影响 | 仅扩展显式新链路/bootstrap；legacy 与普通 shadow 请求不生成 PrimaryDraft，默认 `/trip/plan` 响应不变 |

## 2. 当前代码基线与差异

- legacy `ItineraryPlannerAgent` 通过自由文本生成近似完整行程，混合了日期、景点、酒店、餐饮和建议，无法证明点位来自本次候选集合，也没有严格两次草案边界。
- Stage 2 已定义 `PrimaryItineraryDraft`、主要点位、餐次槽位、空间锚点和建议时长范围，但尚无 Agent 调用、输出对账与业务校验实现。
- Stage 7B 已交付 `PrimaryCandidateSelection`：真实候选 ID 子集、kind、required、constraint IDs 与筛选理由已锁定，但仍是无日期、无顺序的集合。
- 当前没有完整日期覆盖、每日至少一个主要点、所有 selection 恰好使用一次、required 传递、`day_part/meal_role` 互斥、删除排名、餐槽和锚点一致性校验。
- Stage 9A 依赖 PrimaryDraft 的空间顺序选择连续酒店，并为未占用午晚餐检索真实便餐；若本 Stage 提前引用酒店或虚构餐厅 ID，下游无法保持真实性。
- Stage 18 前天气只能被预留和透传，不能改变点位集合、日期、顺序、时段、餐次或理由。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/agents/itinerary_planner_agent.py` | 第一次规划调用的固定 Prompt；只生成 PrimaryDraft，不持有工具或生成事实实体 |
| 新增 | `backend/app/services/primary_draft_service.py` | 构建 Agent 输入、调用 StructuredLLMAdapter、执行跨字段校验与有限重试 |
| 新增 | `backend/app/services/primary_draft_errors.py` | 定义前置业务不可行和草案耗尽等稳定错误码 |
| 修改 | `backend/app/models/itinerary.py` | 补齐 `PrimaryItineraryDraft`、day、item、provisional meal slot 和 anchor 的实现期不变量；不改变 Stage 2 业务语义 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 7B selected 后串行生成 PrimaryDraft并写入内部研究结果 |
| 新增 | `backend/tests/contract/test_itinerary_planner_agent.py` | JSON mode、temperature、schema、Prompt 边界、重试与耗尽测试 |
| 新增 | `backend/tests/unit/services/test_primary_draft_service.py` | 日期、ID、required、约束、类型、时长、餐次、锚点和删除排名测试 |
| 新增 | `backend/tests/integration/test_trip_coordinator_primary_draft.py` | Stage 7B→8 接入、业务短路、天气零影响、模式与上游回归 |

本 Stage 不修改 AmapService、Provider、DestinationResearchAgent、候选池/筛选策略、RouteService、TimelineCalculator、API 路由、legacy Agent、前端类型或页面。

## 4. 输入、输出与错误契约

### 4.1 调用前置与输入快照

- 只允许 `PrimaryCandidateSelectionResult.status=selected` 进入本 Stage；上游 business_infeasible 原样短路，Planner 调用数为 0。
- 输入必须包含 request_id、city、start_date、end_date、travel_days、preferences、normalized accepted constraints、Stage 7B selection，以及按 ID 回填的 Stage 7A 真实候选事实。
- selected_count 小于 travel_days 时，不可能满足每天至少 1 个主要点，确定性返回 `PrimaryDraftBusinessInfeasible(reason_code="insufficient_primary_candidates_for_days")`，Planner 调用数为 0。
- 候选视图只包含 candidate ID、kind、具体名称、地址、坐标、来源 POI 类型、matched preferences、required、constraint IDs、Stage 7B selection reason，以及可用来源事实/质量；不提供 Provider 原始 payload、Key 或 legacy 输出。
- ignored requirements 不进入 Prompt。accepted constraints 只以 normalized 字段和稳定 ID进入；带指定日期的 explicit time 目标必须在该日期安排，未指定日期的目标可由 Planner 分配，最终时刻仍由后续时间轴校验。
- `weather_context` 在请求上下文中预留并由 Coordinator 透传，但 Stage 18 前 `PrimaryDraftService` 不把它写入决策 Prompt或校验分支；天气内容变化不得影响输出。

### 4.2 Agent 调用规范

- 复用 Stage 5 `StructuredLLMAdapter`，使用 JSON mode/`response_format`、`temperature=0.3`；响应字符串直接 `json.loads` 后由 `extra="forbid"` Pydantic 模型校验，禁止正则截取、代码块修复或静默丢字段。
- `ItineraryPlannerAgent.create_primary_draft()` 只能引用输入 candidate ID，不调用 MCPTool、AmapService、DestinationResearchAgent、RouteService 或 TimelineCalculator。
- Agent 负责按空间关联、偏好、accepted 约束和日内逻辑提出日期与顺序，但不得编造距离、耗时、开放时间、票价、酒店、便餐或路线。
- Planner 输出理由和建议时长属于 generated/estimated 决策数据；来源 POI事实仍由 Coordinator 按 ID 回填。

### 4.3 PrimaryDraft 顶层与日期覆盖

`PrimaryItineraryDraft` 至少包含：

- request_id、city、start_date、end_date、travel_days；
- `days`，长度必须等于 travel_days，日期严格覆盖 start_date..end_date，每日一次、顺序递增且不得缺日/越界；
- `scheduled_candidate_count`、`required_count`、`gourmet_count`，由 days 派生并与输入 selection 对账；
- `weather_context` 可选透传引用；Stage 18 前不出现在 Agent 决策字段或理由中。

每个 `PrimaryDraftDay` 包含 date、按实际日内意图排列的 `items` 和 breakfast/lunch/dinner 三个 `provisional_meal_slots`。每日至少 1 个主要点位；“每天 2～3 个”为质量建议而非硬填充目标，不得为了凑数复制或虚构点位。

Stage 7B selection 中每个 candidate ID 必须在整个 draft 中恰好出现一次，draft 不得引用 selection 外 ID。required 与普通点均不能在本次初稿中丢失；后续修复仅能按规则删除 optional。

### 4.4 主要点位 item 契约

每个 item 只引用 candidate ID，并按 kind 使用判别联合：

| kind | 必填字段 | 必须为空/禁止字段 | 建议时长 |
|------|----------|-------------------|----------|
| `attraction` | `day_part=morning|afternoon`、constraint IDs、required、可选 removal rank、reason | `meal_role=null` | 30～240 分钟 |
| `gourmet` | `meal_role=breakfast|lunch|dinner`、constraint IDs、required、可选 removal rank、reason | `day_part=null` | 40～120 分钟 |

- kind、constraint IDs 和 required 必须与 Stage 7A/7B 完全一致；ignored/未知 constraint ID、required 降级或类型互换均非法。
- selection 中全部 item 都需安排一次。required item 的 `removal_rank=null`；每个 optional item 必须有全 draft 唯一的正整数 removal rank，且集合严格等于 `1..optional_count`。数字越小表示全局修复时越先考虑删除，不代表游览顺序。
- reason 去除首尾空白后为 1～200 个 Unicode code point，只解释分日/位置选择，不承载来源事实或最终可执行承诺。
- accepted explicit time 目标必须保留关联 constraint IDs；若约束指定日期，item.date 必须一致。具体时刻窗口在 Stage 11/12B 校验，不由 Agent 声称已验证路线可达。

### 4.5 日内顺序与美食餐次

- 每日 items 数组表达主要点位的日内相对顺序，但不表达最终时刻。gourmet breakfast 必须位于该日所有 attraction 之前，gourmet dinner 必须位于所有 attraction 之后。
- `day_part=morning` 的 attraction 必须全部排在 lunch 边界之前，`day_part=afternoon` 必须全部排在 lunch 边界之后；同一 day_part 内允许多个点并保持 Agent 顺序。
- gourmet lunch 是当日午餐边界：morning attraction 在其前，afternoon attraction 在其后。没有 gourmet lunch 时，provisional lunch slot 在两组 attraction 之间形成虚拟 `LUNCH_SLOT`。
- 每日每个 meal_role 最多一个 gourmet；一个 gourmet 只能占一个角色。美食级餐点是正式餐次，已占用的同餐次不得再要求便餐/住宿侧早餐。
- 当请求显式包含 food 偏好时，N 日 PrimaryDraft 必须包含 1～N 个 gourmet 且每天最多 1 个；当 food=false 时允许 0 个普通 gourmet，但 accepted required gourmet 仍须保留并同样受每天最多 1 个约束。
- 美食级早餐不做固定候选餐窗或营业时间预筛；本 Stage 仅分配 meal_role，实际早餐区间留给 Stage 11/12B。

### 4.6 provisional meal slots 与锚点

每个日期必须恰好输出 breakfast、lunch、dinner 三个槽位，状态沿用 02/03 冻结枚举，只允许 `occupied_by_gourmet|requires_convenience_meal`：

- 对应 meal_role 已有 gourmet 时，状态为 `occupied_by_gourmet` 并引用该日唯一 gourmet candidate ID；不得再携带便餐锚点。
- breakfast 未被 gourmet 占用时使用 `requires_convenience_meal`，但这是冻结 schema 中的“待补餐次”通用名称；不生成 POI 或空间锚点，Stage 9A 必须将早餐确定性映射为 `lodging_side`，不得据此检索或伪造早餐店。
- lunch/dinner 未被 gourmet 占用时使用 `requires_convenience_meal` 并提供 Stage 9A 所需锚点；本 Stage 不填真实便餐 ID。

锚点类型封闭为本次 draft 的 `POI_ID` 或 `DAY_START/LUNCH_SLOT/DAY_END`，不得引用未来酒店或便餐 ID：

- lunch `after_anchor`：最后一个 morning attraction；若无 morning 点则为 `DAY_START`。
- lunch `before_anchor`：第一个 afternoon attraction；若无 afternoon 点则为 `DAY_END`。
- dinner `after_anchor`：最后一个 afternoon attraction；若无 afternoon 点但已有 gourmet lunch，则为该 lunch POI ID；若无 afternoon 点且午餐待 Stage 9A 选择，则必须为 `LUNCH_SLOT`，不得使用 `DAY_END`。
- dinner `before_anchor` 固定为 `DAY_END`。Stage 9A 在选择连续酒店后才把 DAY_START/DAY_END 解析为真实酒店边界，并先选择午餐、再将 LUNCH_SLOT 解析为真实午餐 POI ID 后检索晚餐。
- POI anchor 必须引用同一天 items 中存在且位于相应槽位邻接侧的 candidate ID；跨日、未知 ID或顺序矛盾均非法。

### 4.7 确定性输出校验顺序

每次 Agent 响应必须依次通过：

1. JSON、`extra="forbid"` schema 与 request_id/请求日期对账；
2. days 数量、连续日期、每日非空与派生计数校验；
3. selection ID 全集相等、全局唯一、kind/required/constraint IDs 一致；
4. explicit time 指定日期与 item 日期一致；
5. attraction/gourmet 判别字段和建议时长范围；
6. required 无 removal rank、optional removal rank 为全局连续唯一排列；
7. gourmet 总数、每天最多一个、meal_role 唯一与日内位置；
8. 每日三个 meal slots 完整，occupied 状态与 gourmet 一一对应；
9. lunch/dinner anchors 存在、同日、方向和 `LUNCH_SLOT` 特例正确；
10. weather_context 未影响决策字段，且输出不含酒店、便餐、路线、时刻、费用或最终 TripPlan 字段。

任何一项失败均拒绝整份 draft并进入有限重试，不允许保留部分日期或让 TimelineCalculator补默认值。

### 4.8 有限重试、错误与日志

- 首次失败后最多重试 2 次，总调用上限 3；schema/JSON、日期、ID、约束、时长、排名、餐次、槽位、锚点或越界字段错误均可重试。
- 重试使用相同输入快照、temperature 和 schema，只追加稳定脱敏校验码；不得回显完整 Prompt、候选事实、用户原文或上次完整输出。
- 任一合法响应立即停止。三次均失败时抛 `PrimaryDraftError(error_code="primary_draft_retry_exhausted")`，属于核心规划技术错误；不得转 ignored、business_infeasible、partial draft 或虚构 fallback。
- 输入 selection 数量小于天数等调用前可证明的不可能条件返回业务不可行；LLM 超时、认证、限流、非法响应和重试耗尽保持技术错误类别，两者不得互换。
- 日志按 request_id 记录 stage=`primary_draft`、attempt、day/item/required/gourmet count、validation/error code、status 和 duration；不得记录 Key、Prompt、自由文本、完整候选、完整 Agent 输出或 POI事实。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或功能开关；temperature=0.3、最多重试 2 次、时长范围、餐次枚举和锚点规则为受测试锁定的领域策略。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求不运行 Coordinator；显式 shadow 验证可执行 Stage 8；new 仍为 reserved。
- 复用现有 LLM 配置和 StructuredLLMAdapter 注入；ItineraryPlannerAgent 不读取 Settings、不持有外部工具。

## 6. 实施步骤

1. 复核并实现 Stage 2 PrimaryDraft/day/item/meal slot/anchor 严格模型及序列化正反例。
2. 建立 ItineraryPlannerAgent 第一次调用 Prompt，固定 temperature=0.3、JSON schema、ID 白名单和两次草案职责边界。
3. 实现输入快照与 deterministic preflight，覆盖上游状态、selected_count 与 travel_days、accepted constraints 和天气零影响。
4. 实现日期、selection 全覆盖、required/constraint、判别字段、时长和 removal rank 校验。
5. 实现美食 1～N、每天最多一个、meal_role、日内顺序及三餐槽位一致性校验。
6. 实现 lunch/dinner anchor 推导验证，锁定无下午点时 `LUNCH_SLOT` 正反例和未来酒店/便餐 ID 禁止规则。
7. 实现最多 2 次重试、稳定脱敏错误反馈和 `primary_draft_retry_exhausted` 技术错误。
8. 接入 TripCoordinator 并补 Stage 7B→8 集成回归；执行 Stage 1～8 测试与前端构建，归档证据。

## 7. 明确不做

- 不重新筛选、添加或删除 Stage 7B selection；初稿必须安排全部 selection，后续可选点修复由 Stage 12/13 承担。
- 不选择酒店、住宿侧早餐实体或便餐，不解析 DAY_START/DAY_END 为酒店 ID；属于 Stage 9A。
- 不生成完整 ItineraryDraft，不加入酒店和便餐到活动序列；属于 Stage 9B。
- 不调用路线、不计算最终时刻、餐窗、缓冲、总时长或费用，不以估算距离宣称可执行。
- 不校验美食级早餐营业时间，不将 unavailable 可选事实补成默认值。
- 不让天气影响任何决策；Stage 18 接入时必须按“天气先筛选、再重建 PrimaryDraft”重新评审调用边界。
- 不修改普通 API、前端或 legacy 链路，不开放 new 模式，不删除 legacy fallback。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S8-T01 | contract | 合法 1～3 日 PrimaryDraft | JSON mode、temperature=0.3、extra=forbid，日期完整 |
| S8-T02 | contract | 输出酒店/便餐/路线/时刻/费用/TripPlan 字段 | schema 拒绝并重试 |
| S8-T03 | unit | selected_count < travel_days | 零 LLM，business_infeasible/insufficient_primary_candidates_for_days |
| S8-T04 | unit | 缺日、重复日、日期乱序或越界 | 全部拒绝，不形成 partial draft |
| S8-T05 | unit | 某日零主要点 | 拒绝，每日必须至少 1 个 |
| S8-T06 | unit | selection ID 遗漏、重复、未知 | 均拒绝；scheduled ID 集合必须与 selection 完全相等 |
| S8-T07 | unit | kind/required/constraint IDs 被修改 | 拒绝并给稳定校验码 |
| S8-T08 | unit | explicit time 指定日期目标 | 仅目标安排在指定日期时通过，时刻可执行性留后续 |
| S8-T09 | unit | attraction 字段 | day_part 必填、meal_role=null、时长 30～240 |
| S8-T10 | unit | gourmet 字段 | meal_role 必填、day_part=null、时长 40～120 |
| S8-T11 | unit | 时长缺失或越界 | 非法输出重试，Calculator 不补默认值 |
| S8-T12 | unit | required removal rank | required 必须为 null，任何数字均拒绝 |
| S8-T13 | unit | optional removal ranks | 必须全局唯一且恰为 1..optional_count，缺口/重复拒绝 |
| S8-T14 | unit | food=true 的 N 日草案 | gourmet 总数 1～N，每天最多 1 个 |
| S8-T15 | unit | food=false + required gourmet | required gourmet 保留且每天最多 1 个，不要求额外普通 gourmet |
| S8-T16 | unit | 两个 gourmet 同日或重复 meal_role | 均拒绝 |
| S8-T17 | unit | gourmet breakfast/lunch/dinner 日内位置 | 分别位于早餐首段、午间边界、晚餐末段 |
| S8-T18 | unit | attraction day_part 与 items 顺序 | morning 全在午餐前、afternoon 全在午餐后 |
| S8-T19 | unit | 三餐 slots 完整性 | 每日恰好 breakfast/lunch/dinner，无缺失/重复 |
| S8-T20 | unit | occupied_by_gourmet | slot 引用同日对应 meal_role 唯一 ID，且无便餐锚点 |
| S8-T21 | unit | 非 gourmet breakfast | requires_convenience_meal 且无 POI/锚点，供 9A 映射 lodging_side |
| S8-T22 | unit | lunch anchors | last morning/DAY_START 与 first afternoon/DAY_END 正反例通过 |
| S8-T23 | unit | dinner 有下午点 | after_anchor 为最后 afternoon，before_anchor=DAY_END |
| S8-T24 | unit | 无下午点且午餐待选 | dinner_after_anchor 必须为 LUNCH_SLOT，使用 DAY_END 拒绝 |
| S8-T25 | unit | 无下午点但已有 gourmet lunch | dinner_after_anchor 为 gourmet lunch ID，不用 LUNCH_SLOT |
| S8-T26 | unit | anchor 未知、跨日或方向错误 | 全部拒绝，不引用未来酒店/便餐 ID |
| S8-T27 | contract | 首次成功、第三次成功 | 调用数分别为 1、3，重试输入快照不变 |
| S8-T28 | contract | 三次校验失败 | 抛 primary_draft_retry_exhausted，不返回部分草案 |
| S8-T29 | contract | LLM 技术异常耗尽 | 保持技术错误，不转换业务失败、ignored 或 fallback |
| S8-T30 | integration | 7B selected → PrimaryDraft | Coordinator 保存草案，所有 ID 可回填同一事实实体 |
| S8-T31 | integration | 上游 business_infeasible | Planner 调用数 0，状态与 reason 原样短路 |
| S8-T32 | integration | weather_context 内容变化 | Stage 18 前 Agent 决策输入与 PrimaryDraft 完全一致 |
| S8-T33 | contract | Agent 依赖边界 | 无 MCPTool/AmapService/RouteService，不导入 legacy planner |
| S8-T34 | regression | legacy/shadow/new 模式 | 普通 legacy/shadow Planner 调用数 0，new 仍 planner_mode_not_ready |
| S8-T35 | regression | Stage 1～7B 与前端 | 上游测试和生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_primary_draft_service.py backend/tests/contract/test_itinerary_planner_agent.py backend/tests/integration/test_trip_coordinator_primary_draft.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实 LLM。Stage 8 必测用例数为 35，失败数必须为 0。

## 9. 人工验收

不适用：Stage 8 不接入普通 API/UI；日期、顺序、ID、required、餐次、slot、anchor、重试和天气零影响均由 FakeLLM 与固定候选 fixture 自动化覆盖。显式 shadow 人工入口沿用 Stage 5，不重复建立页面验收。

## 10. 量化退出指标

- [ ] S8-T01～S8-T35 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] days 精确覆盖请求 1～3 日且每天至少 1 个主要点；缺日、重复日、空日和越界日通过数为 0。
- [ ] Stage 7B selected ID 安排率为 100%，每个 ID 恰好出现一次；未知、重复或遗漏 ID 通过数为 0。
- [ ] required、kind、constraint IDs 与上游一致率 100%；required removal rank 非空和 optional rank 非连续的通过数为 0。
- [ ] attraction/gourmet 判别字段正确率 100%，建议时长分别严格在 30～240/40～120 分钟。
- [ ] food=true 时 gourmet 总数为 1～N 且每天最多 1 个；food=false 时 required gourmet 保留，普通美食下限不被错误启用。
- [ ] 每日三个 meal slots 数量精确为 3，occupied 状态与 gourmet 一一对应；同餐次重复安排数为 0。
- [ ] lunch/dinner anchor 可由同日 items 确定性复核；无下午点且午餐未选时 LUNCH_SLOT 使用率 100%，误用 DAY_END 通过数为 0。
- [ ] 输出中的酒店 ID、便餐 ID、路线、最终时刻、费用和 TripPlan 字段数均为 0。
- [ ] 一次成功调用数为 1、第三次成功和耗尽均为 3；耗尽全部返回 primary_draft_retry_exhausted 技术错误，不产生 partial draft或 fallback。
- [ ] Stage 18 前天气变化导致决策输入或草案变化的用例数为 0；ignored requirements 进入 Prompt/草案数为 0。
- [ ] legacy/shadow 普通请求行为不变，new 仍 reserved；Stage 1～7B 回归与前端构建通过。
- [ ] 03 Stage 8 每条退出条件及 02 PrimaryDraft/餐次/锚点规则均映射到测试/证据 ID，新增未追踪 TODO 数为 0。

## 11. 迁移与回滚

- Stage 8 只在显式新链路内部增加 PrimaryDraft；默认用户流量仍为 legacy，无持久化数据迁移。
- 回滚时移除 Coordinator 的 PrimaryDraft 调用、ItineraryPlannerAgent 第一次调用入口、service/error 及本 Stage模型细化，恢复 Stage 7B selection 作为 bootstrap 末端；Stage 1～7B 行为保留。
- 回滚不得删除 StructuredLLMAdapter、DestinationResearchAgent、PrimaryCandidatePool 或 PrimaryCandidateSelection。
- 回滚后 Stage 1～7B 全量测试、legacy/shadow 模式测试和前端生产构建必须通过。
- Stage 9A 以后已依赖 PrimaryDraft 时必须按依赖逆序回滚，不能单独移除草案或改变 anchor 语义。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_8/test-summary.md` | 提交标识、三条命令、35 个用例结果、离线声明和失败数 |
| PrimaryDraft 契约矩阵 | `doc/opt_strategy/evidence/stage_8/primary-draft-contract.md` | 日期、ID、required、类型、约束、时长、排名和测试 ID |
| 餐次与锚点矩阵 | `doc/opt_strategy/evidence/stage_8/meal-slot-anchor-matrix.md` | 三餐状态、gourmet 角色、day_part、LUNCH_SLOT 正反例 |
| 重试与错误矩阵 | `doc/opt_strategy/evidence/stage_8/retry-error-matrix.md` | preflight、1/3/耗尽、技术/业务分流和稳定错误码 |
| 边界审计 | `doc/opt_strategy/evidence/stage_8/stage-boundary-check.md` | 7B 输入、8 输出及未越界进入 9A/9B/路线/时间轴的检查 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_8/rollback-check.md` | Coordinator 恢复点、Agent 入口撤销、依赖逆序、命令与结果 |

## 13. Ready/Done 复核

当前设计状态：**工作包已细化，待 Stage 6～8 第二次跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 2、5、7B 实际 Done；审计时重点复核 Stage 6→7A→7B→8 的 ID/required/constraint 全链传递、food 与餐次规则、PrimaryDraft 到 Stage 9A 的 slot/anchor 边界，以及核心输出错误不得复用 parse_failed 例外。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
