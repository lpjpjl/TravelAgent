# Stage 11：TimelineCalculator

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 11 |
| 目标 | 基于完整活动顺序与已选真实路线，以 `Asia/Shanghai` 分钟精度确定性生成只读日时间轴、独立 buffer/wait 事件和可修复问题 |
| 对应优化项 | P0-7 行程深度关联化、P0-8 可执行时间轴信息 |
| 前置依赖 | Stage 10B |
| 后续阻断 | 未通过时不得开始 Stage 12A、12B、13A～13D、14、16、18、20、28 |
| 默认流量影响 | 无；仅扩展显式新链路内部结果，legacy/shadow 普通请求与 `new` reserved 行为不变 |

## 2. 当前代码基线与差异

- 当前后端无时间轴模型或计算器；legacy Agent 以自由文本写出建议时段，无法关联真实路线、酒店边界或餐饮实体。
- Stage 9B 已锁定完整 `ItineraryDraft` 的活动顺序、三餐、起止酒店和建议时长；Stage 10A/10B 已为每个必要段选择真实 `RouteSegment`，但均不生成最终时刻。
- 当前前端结果页只按景点列表显示，不含可验证的出发、交通、缓冲、到达和结束酒店事件；展示改造属于 Stage 14。
- 目前没有固定餐窗、首末日边界、住宿侧早餐、公交刷新或超窗问题的确定性语义。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/timeline.py` | 固化带 `+08:00` 的 timeline event、day、计算结果、问题和修复候选模型 | naive datetime、乱序事件与不合法状态组合拒绝 |
| 新增 | `backend/app/services/timeline_calculator.py` | 纯函数前向排程、餐窗/wait/buffer、首末日边界、问题生成 | 不导入 Agent、Provider、BudgetCalculator 或 API |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在路线 ready 后调用 Calculator；仅编排一次 transit 刷新和重算 | 不修复活动/餐厅/酒店，不组装 TripPlan |
| 新增 | `backend/tests/unit/services/test_timeline_calculator.py` | 覆盖排程、餐窗、buffer、住宿、问题与稳定排序 | 固定时区/时钟，不访问外部服务 |
| 修改 | `backend/tests/unit/models/test_route_timeline.py` | 覆盖 timeline 模型、事件连续性和问题序列化 | datetime 全为 `Asia/Shanghai` offset |
| 新增 | `backend/tests/integration/test_trip_coordinator_timeline.py` | 覆盖 Stage 10B→11、transit 刷新、上游短路与模式回归 | 使用 FakeRouteService，不访问公网 |

本 Stage 不修改 RouteService、RoutePolicy、RouteSelection、ItineraryDraft、餐饮/酒店选择、预算、Validator、API 路由、前端或 legacy Agent。它不执行任何修复；Stage 13B～13D 才消费问题。

## 4. 输入、输出与错误契约

### 4.1 输入与前置

`TimelineCalculator.calculate()` 只接收同一 request_id 的完整 `ItineraryDraft`、`RoutePlanningResult.ready`、请求日期范围、活动/酒店真实事实引用及每项建议时长：

- days、活动、酒店边界与 RouteSegment requirement 必须一一对账；未知 ID、缺失必要段、跨日段、末日返店段或 `lodging_side` 早餐被作为外出路线端点均为 `invalid_timeline_input` 技术校验错误，Calculator 不地理编码或补路线；
- Stage 10B 的 needs_repair、上游 business_infeasible/技术错误原样短路，Calculator 调用次数为 0；
- 输入不含用户原文、Prompt、完整 Provider payload 或价格；天气在 Stage 18 前不参与排程分支。

### 4.2 事件与前向排程

每个 `TimelineDay` 按实际开始时刻、同刻事件优先级和稳定 ID 排序，事件只允许 `departure`、`route`、`buffer`、`wait`、`activity`、`arrival_hotel`、`breakfast_lodging_side_note`：

1. `lodging_side` 早餐仅产生离店前完成的 note，首段 `departure` 为 09:00；美食级早餐首段 `departure` 为 08:00，作为真实 `activity`。
2. 每条真实 RouteSegment 生成 route（来源距离/耗时）后紧接至少 30 分钟独立 buffer；buffer 不得被 wait、活动或路线吞并。
3. attraction 使用其 Stage 8/9B 建议时长；在外早餐/午餐/晚餐均不少于 40 分钟。Calculator 不改写时长。
4. 午餐选择不早于 12:00 的最早可行开始，结束不得晚于 14:00；普通日晚餐选择不早于 18:00 的最早可行开始，结束不得晚于 20:00；`day.date == request.end_date` 的晚餐窗口为 17:00—18:00。为等待窗口开始可插入独立 wait。
5. morning attraction 必须在实际午餐开始前结束，afternoon attraction 必须在午餐结束后开始；gourmet 按 meal_role 作为对应餐次本身。Calculator 不移动活动，不能满足时记录问题而非调整顺序。
6. 非末日最后 route+buffer 后生成 `arrival_hotel`，最晚 21:00；末日以晚餐结束为旅程终点，不生成结束酒店、返店 route 或 arrival_hotel。

所有 datetime 均为 `Asia/Shanghai` 且精确到分钟。partial 语义尚未在本 Stage 构造；日期是否末日只能由 `day.date == request.end_date` 判断，不能以当前数组末项判断。

### 4.3 公交一次刷新

- 对每条 transit route，Calculator 将其计算出的实际 departure 与 Stage 10A `query_departure_at` 比较；绝对差 `>30` 分钟且 refresh_count=0 时，返回唯一 `transit_time_refresh_required` 问题，不返回可供最终组装的 timeline。
- Coordinator 只可调用相同端点的 `RouteService.refresh_transit_segment()` 一次，替换该段后从头重新调用 Calculator；刷新后再次超过阈值但 refresh_count=1 时继续按真实新路线排程，不触发第二次刷新。
- refresh 技术失败或 `transit_refresh_limit_reached` 原样传播；不得改用直线、其他交通方式或伪造时刻。

### 4.4 结果与问题

`TimelineCalculationResult` 为内部对象：

| 状态 | 输出 | 语义 |
|---|---|---|
| `scheduled` | 全部 TimelineDay、事件与日汇总 | 所有固定窗口和日边界满足；无问题 |
| `requires_repair` | 完整的已计算时间轴与稳定排序问题列表 | 活动顺序不变，但存在超窗/边界/午餐分段等不可执行事实 |

- 问题代码仅为 `meal_window_violation`、`day_part_lunch_conflict`、`day_end_deadline_exceeded`、`final_day_end_deadline_exceeded`、`daily_duration_exceeded`、`transit_time_refresh_required`；每项含日期、活动/route ID、冲突区间、稳定原因及按 removal_rank 升序的 optional 修复候选。
- 非末日住宿侧早餐最长离店至抵达结束酒店 12 小时；美食级早餐最长 13 小时；末日 18:00 为硬上限。超限仅产生问题，不压缩路线、buffer、用餐或活动时长。
- Calculator 不决定 complete/partial/failed、不删除候选、不替换路线/餐厅/酒店；Stage 12B 继续做正式质量闸门，Stage 13B～13D 决定修复。
- 日志只记录 request_id、stage=`timeline`、日期、事件/问题计数、refresh_count、稳定 code 和耗时；不得记录地址、完整坐标/polyline、自由文本或完整行程。

## 5. 配置与功能开关

- 不新增 Settings、环境变量或用户流量开关。`Asia/Shanghai`、30 分钟 buffer、午餐/晚餐窗口、首末日上限为 Stage 2/03 已冻结且受测试锁定的领域规则。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求不执行本 Stage，new 在 Stage 16 前仍为 `planner_mode_not_ready`。
- Calculator 通过参数接收时钟/时区，不读取 Settings；公交刷新上限沿用 Stage 10A 的每段最多一次规则。

## 6. 实施步骤

1. 补齐 timeline event、day、结果和问题模型，锁定时区、事件类型、问题码和只读语义。
2. 实现输入对账、外出活动/路线配对及住宿侧早餐排除，拒绝缺路段和跨日错误。
3. 实现纯函数前向排程：route→buffer→wait（仅必要时）→activity，保留所有原始时长与顺序。
4. 实现三餐窗口、morning/afternoon 午餐边界、非末日酒店抵达、末日结束和日时长问题。
5. 接入 Calculator 的 transit 差异检测；Coordinator 仅编排一次刷新和完整重算。
6. 用固定 routes/drafts 验证结果、问题排序和无副作用；执行 Stage 1～11 回归与前端构建，归档证据。

## 7. 明确不做

- 不调用高德、选择/替换路线、改变交通方式或计算缓存。
- 不校验餐厅营业时间、饮食/accepted constraint 追溯、路线真实性或预算；属于 Stage 12A/12B。
- 不替换餐厅/酒店、重排/删点、构造 partial/failed 或 TripPlan；属于 Stage 13A～13D。
- 不实现时间轴 UI、地图联动、SSE 或前端状态；属于 Stage 13E、14、20、28。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S11-T01 | unit | lodging_side / gourmet 早餐首段 | 分别 09:00/08:00；前者无外出 activity/route |
| S11-T02 | unit | route、buffer、activity 顺序 | 每条路线后恰一独立 ≥30min buffer，时长不被吞并 |
| S11-T03 | unit | 午餐 wait 与 40min 时长 | 12:00 前等待、完整落在 12:00—14:00 |
| S11-T04 | unit | 普通日/末日晚餐 | 分别完整落在 18:00—20:00 / 17:00—18:00 |
| S11-T05 | unit | morning/afternoon 与午餐 | 不跨午餐时 scheduled；跨越时 day_part_lunch_conflict |
| S11-T06 | unit | 非末日结束酒店/末日终点 | 非末日 arrival_hotel ≤21:00；末日无返店且晚餐结束 |
| S11-T07 | unit | 12/13 小时日上限 | 按早餐类型产生 daily_duration_exceeded，不压缩事实 |
| S11-T08 | unit | 1～3 日期与 partial 样本 | 只按 request.end_date 判末日，partial DayK 仍用非末日规则 |
| S11-T09 | unit | 缺段、未知 ID、lodging_side 路线 | invalid_timeline_input，计算事件数为 0 |
| S11-T10 | unit | 多个超窗问题 | 完整 timeline 保留；问题按日期/活动/优先级稳定排序 |
| S11-T11 | unit | optional removal candidates | 仅引用 optional rank，required/三餐/酒店不得进入候选 |
| S11-T12 | unit | transit 偏差 30/31 分钟 | 30 不刷新；31 返回 transit_time_refresh_required |
| S11-T13 | integration | 一次 transit 刷新 | Coordinator 刷新一次后重算；第二次不请求 |
| S11-T14 | integration | refresh 技术失败 | 原样技术错误，无直线/交通替换 |
| S11-T15 | contract | timeline 模型 | +08:00、分钟精度、事件顺序和状态互斥严格校验 |
| S11-T16 | integration | Stage 10B→11 happy path | 保存 scheduled 内部 timeline，供 12A/12B 使用 |
| S11-T17 | integration | 上游 needs_repair/技术错误 | Calculator/refresh 调用均为 0，状态原样短路 |
| S11-T18 | regression | legacy/shadow/new 与上游 | 普通旧流量调用数为 0；new reserved；Stage 1～10B/前端不回退 |

```text
python -m pytest backend/tests/unit/models/test_route_timeline.py backend/tests/unit/services/test_timeline_calculator.py backend/tests/integration/test_trip_coordinator_timeline.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、真实高德或真实 LLM。Stage 11 必测用例数为 18。

## 9. 人工验收

不适用：本 Stage 无 API/UI 接入；时间轴算法、公交刷新和问题均由固定时钟、FakeRouteService 和离线 fixture 覆盖。Stage 14 再进行可视时间轴人工验收。

## 10. 量化退出指标

- [ ] S11-T01～S11-T18 全部通过，失败数为 0，三条命令退出码为 0。
- [ ] 所有 timeline datetime 带 `+08:00` 且分钟精度；naive datetime 接受数为 0。
- [ ] 每条真实路线后的独立 buffer 数为 1，少于 30 分钟或被 wait 合并的 buffer 数为 0。
- [ ] 午餐、普通日晚餐、末日晚餐窗口越界的 scheduled 结果数为 0；末日返店段数为 0。
- [ ] 31 分钟以上 transit 偏差的刷新次数为 1，单段第二次刷新成功数为 0。
- [ ] Calculator 修改活动顺序、事实时长、路线、餐厅、酒店或生成 partial/TripPlan 的次数为 0。
- [ ] 默认测试外部调用数为 0；日志/证据中敏感数据命中数为 0；Stage 1～10B 回归与前端构建通过。

## 11. 迁移与回滚

- Stage 11 只在显式新链路增加内部 timeline；无用户数据、默认 API 或前端迁移。
- 回滚先移除 Coordinator 的 calculator/refresh 编排，再移除 calculator、模型增量和测试；保留 Stage 10A/10B 路线语义与 ItineraryDraft。
- Stage 12A 以后依赖时按 13D→12B→12A→11 逆序回滚。回滚不得以文本时段、直线或压缩 buffer 取代真实路线时间。
- 回滚后执行 Stage 1～10B 全量、模式回归和前端构建，期望退出码为 0。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_11/test-summary.md` | 提交标识、三条命令、18 用例、离线声明和失败数 |
| 排程矩阵 | `doc/opt_strategy/evidence/stage_11/timeline-schedule-matrix.md` | 早餐、餐窗、buffer、酒店边界与测试 ID |
| 问题矩阵 | `doc/opt_strategy/evidence/stage_11/timeline-issue-matrix.md` | 问题码、冲突区间、removal rank 与测试 ID |
| 公交刷新矩阵 | `doc/opt_strategy/evidence/stage_11/transit-refresh-matrix.md` | 0/1/2 次刷新、重算和错误边界 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_11/rollback-check.md` | 入口撤销、依赖逆序、命令和结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 10B 实际 Done，并在开工前复核 RouteSegment、ItineraryDraft、transit refresh 与 Stage 12A/12B 的消费边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
