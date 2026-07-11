# Stage 12B：TimelineValidator

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 12B |
| 目标 | 在最终组装前对事实引用、路线、时间轴、住宿三餐、accepted constraints 与预算输入执行确定性质量闸门，输出稳定且可修复的问题列表 |
| 对应优化项 | P0-7 行程深度关联化（确定性校验部分）、P0-8 可执行时间轴 |
| 前置依赖 | Stage 11、Stage 12A |
| 后续阻断 | 未通过时不得开始 Stage 13A～13D、14、16 |
| 默认流量影响 | 无；仅扩展显式新链路内部校验，legacy/shadow 普通请求与 `new` reserved 行为不变 |

## 2. 当前代码基线与差异

- 当前没有最终质量闸门；legacy Agent 解析到任意 JSON 即可能返回“成功”计划，未知 POI、空路线、虚构价格、餐次重复和时间冲突没有统一语义。
- Stage 11 提供不修改草案的 timeline/问题，Stage 12A 提供预算及 unavailable 项；二者尚未核对真实来源、accepted constraints、完整日规则或必要项。
- 当前 API 将多数故障统一包装为 500；本 Stage 只定义内部问题，不提前映射 HTTP 或展示 UI（Stage 15）。
- 路线、餐饮、酒店和约束均已有稳定 ID/来源模型，但没有跨对象的完整性与可执行性检查。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/timeline.py` | 增加 `ValidationIssue`、severity、repair scope 与 ValidationResult 严格模型 | unknown/重复问题 ID、非法 scope 与 valid+blocking 并存拒绝 |
| 新增 | `backend/app/services/timeline_validator.py` | 纯函数执行来源、路线、时间、住宿/三餐、约束和预算输入校验 | 不调用 Agent、Provider、RouteService 或 Calculator |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 11/12A 结果后调用 Validator 并保存内部 ValidationResult | 不组装 TripPlan、不做修复或 partial |
| 新增 | `backend/tests/unit/services/test_timeline_validator.py` | 覆盖每类问题、warning、排序和无副作用 | 固定 DTO，不访问外部服务 |
| 修改 | `backend/tests/unit/models/test_route_timeline.py` | 覆盖 ValidationIssue/Result schema 与稳定 JSON | repair scope、severity、references 严格校验 |
| 新增 | `backend/tests/integration/test_trip_coordinator_validation.py` | 覆盖 Stage 11/12A→12B、上游短路和模式回归 | Validator 为 13A 保存唯一内部结果 |

本 Stage 不修改 TimelineCalculator、BudgetCalculator、RouteService、候选/酒店/餐厅选择或任何 Agent；不映射 API 错误、不显示问题、不替换路线/餐厅/酒店、不重排删点。对应工作分别属于 Stage 13B～13D、14、15。

## 4. 输入、输出与错误契约

### 4.1 输入与输出

`TimelineValidator.validate()` 接收同一 request_id 的 request、normalized accepted constraints、来源 POI/酒店/路线事实、ItineraryDraft、RoutePlanningResult.ready、TimelineCalculationResult 与 BudgetCalculationResult：

- 上游技术错误、路线 needs_repair、transit refresh pending 或 input 对象 request_id 不同均原样短路，Validator 调用数为 0；
- 所有 cross-reference 必须可回填到本次来源对象；未知 ID、重复实体或不一致 source metadata 为 blocking `unknown_source_reference`/`source_integrity_violation`；
- 输出 `ValidationResult(status=valid|invalid, issues, warnings)`。valid 仅允许 issues 为空；warnings 只表示来源字段 unavailable，不能掩盖 blocking/reparable issue；
- issues 按 `(date, day_index, activity_index, severity_rank, error_code, stable_ref)` 排序，Calculator/Validator 都不得改变输入对象。

### 4.2 校验域与问题码

| 域 | 必测规则 | blocking/reparable code |
|---|---|---|
| 来源与结构 | 真实 POI/酒店/路线 ID、GCJ-02 坐标、required 标识和 Agent 禁止字段 | `unknown_source_reference` / `source_integrity_violation` |
| 路线 | 每个外出相邻端点恰一真实段、polyline/距离/耗时、walking 与 transit 接驳上限 | `missing_required_route` / `route_mode_ineligible` |
| 时间 | 事件连续、route 后独立 buffer、餐饮 ≥40min、餐窗、day_part、日边界 | `timeline_conflict` / `meal_window_violation` / `day_end_deadline_exceeded` |
| 住宿和三餐 | lodging_count、连续酒店、末日边界、每日唯一三餐及真实餐饮 POI | `lodging_continuity_violation` / `required_meal_missing` / `meal_source_invalid` |
| 营业时间 | 有来源时完整覆盖最终用餐区间；缺来源仅保留核实提示 | `opening_hours_conflict` / warning `opening_hours_unavailable` |
| accepted constraints | required、explicit time、dietary exclusion、mobility 约束、ignored 零效果 | `accepted_constraint_unsatisfied` / `explicit_time_violation` / `route_mode_ineligible` |
| 预算输入 | lodging_count、计价单位、known total/unavailable 分离与来源/规则依据 | `budget_input_inconsistent` |

- `meal_role=lunch` gourmet 是午餐本身，不适用“非餐饮点跨午餐”规则；便餐与美食级餐点必须来自本次具体来源 POI，类别名、页面文案或同餐次重复均失败。
- `day.date == request.end_date` 才使用末日晚餐 17:00—18:00、无结束酒店/返店段；partial DayK<K 请求总天数时仍按非末日、必须抵达 `lodging[K]`。本 Stage 只验证传入 partial，不创建或截取它。
- required POI、explicit time 隐含 required、酒店和三餐均不可成为 deletion candidate；指定日期/时段无法满足时报告 `explicit_time_violation`，供 Stage 13C 在完整日期范围重排；其他 accepted constraint 无法满足为 blocking，Stage 13D 必须 failed 而非 partial。
- dietary exclusion 仅检查系统选择的真实 gourmet/convenience 餐饮 POI，不检查 lodging_side 早餐；ignored clause 不得出现在查询、候选、草案、路线、时间轴或选择理由中。

### 4.3 修复边界

- `route_mode_ineligible` 的 repair_scope 为 `route`，供 Stage 13B 从已过滤真实候选换路；
- `opening_hours_conflict`、餐饮锚点/窗口问题的 scope 为 `meal`，供 Stage 13B 替换同级真实餐厅；
- 酒店边界、必要路线缺失导致的日不可行 scope 为 `lodging`，供 Stage 13B 重建相邻日；
- `explicit_time_violation` 的 scope 为 `primary_draft`；`accepted_constraint_unsatisfied` 保持 fatal；普通超窗 optional 候选 scope 为 `optional_removal`，分别留给 Stage 13C/13D；
- Validator 仅报告 scope 和引用，绝不挑选替代 ID、删除活动或声明 partial/failed。

## 5. 配置与功能开关

- 本 Stage 不新增 Settings、环境变量或开关。有效步行上限、餐窗、buffer、时区、人数/房间假设沿用前置 Stage 的冻结规则。
- 沿用 `TRIP_PLANNER_MODE`；普通 legacy/shadow 调用为 0，new 在 Stage 16 前仍 reserved。
- Validator 不读取 Settings 或时间；规则均由输入模型和纯函数常量提供。

## 6. 实施步骤

1. 定义 ValidationIssue/Result、severity、repair scope、warning 与稳定排序模型。
2. 实现来源/ID/坐标/禁止 Agent 字段和路线完整性检查，锁定 walking/transit 上限。
3. 实现 timeline、buffer、餐窗、day_part、首末日、酒店链和三餐实体检查。
4. 实现营业时间覆盖/缺失 warning、accepted/ignored constraint 与预算输入一致性检查。
5. 在 Coordinator 保存 ValidationResult；只为后续 Stage 暴露问题，不修复或组装。
6. 执行定向、Stage 1～12B 回归和前端构建，归档问题码/边界/回滚证据。

## 7. 明确不做

- 不生成时间轴、预算、路线、候选或营业时间解析。
- 不把合法业务问题映射为 HTTP/页面错误，不处理 stale cache；属于 Stage 15/17。
- 不替换任何实体、重排/删除点位、构造 partial/failed 或 TripPlan；属于 Stage 13A～13D/16。
- 不实现时间轴/预算 UI 或地图联动；属于 Stage 14/20。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S12B-T01 | unit | 合法完整 1～3 日输入 | status=valid，无 issues，可有 unavailable warnings |
| S12B-T02 | unit | 未知/重复 POI、酒店、路线 ID | source integrity blocking，不修改输入 |
| S12B-T03 | unit | 缺段/直线/步行超限/transit 接驳超限 | missing route 或 route_mode_ineligible，scope=route |
| S12B-T04 | unit | buffer、餐饮时长、午晚餐窗口 | 对应稳定 timeline/meal issue |
| S12B-T05 | unit | morning/afternoon 跨午餐与 gourmet lunch | 前者失败，后者不误报 |
| S12B-T06 | unit | 非末日酒店、末日晚餐/返店、partial DayK | 正确按 request.end_date 判定 |
| S12B-T07 | unit | 三餐缺失/重复/类别文案/非本次 POI | required_meal_missing 或 meal_source_invalid |
| S12B-T08 | unit | lodging_count/连续链与预算单位 | lodging/budget input 问题稳定 |
| S12B-T09 | unit | 有营业时间覆盖/冲突/缺失 | 冲突为 issue；缺失仅 warning+核实提示 |
| S12B-T10 | unit | required、explicit time、dietary、mobility、ignored | accepted 不满足 blocking；ignored 零效果 |
| S12B-T11 | unit | required/酒店/三餐进入 removal candidates | 拒绝并标 source integrity violation |
| S12B-T12 | unit | 多问题排序与 repair scope | 排序稳定，route/meal/lodging/primary/optional 对应正确 |
| S12B-T13 | contract | ValidationResult JSON | valid+issues、warning 形状、未知 scope/severity 均拒绝 |
| S12B-T14 | integration | Stage 11+12A→12B happy path | Coordinator 保存 valid result，供 13A 使用 |
| S12B-T15 | integration | timeline requires_repair/预算 unavailable | 前者 invalid；后者 warning 不阻断合法日 |
| S12B-T16 | integration | 上游 short-circuit | Validator 调用数 0，状态原样传递 |
| S12B-T17 | regression | legacy/shadow/new 与上游 | 普通旧流量调用数 0；Stage 1～12A/前端不回退 |

```text
python -m pytest backend/tests/unit/models/test_route_timeline.py backend/tests/unit/services/test_timeline_validator.py backend/tests/integration/test_trip_coordinator_validation.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、真实高德或真实 LLM。Stage 12B 必测用例数为 17。

## 9. 人工验收

不适用：本 Stage 无 API/UI，所有问题码、warning、排序和 repair scope 均由离线 fixture/Fake DTO 覆盖。Stage 14/15 才分别人工验证展示与错误文案。

## 10. 量化退出指标

- [ ] S12B-T01～S12B-T17 全部通过，失败数为 0，三条命令退出码为 0。
- [ ] valid 结果的 blocking issue 数为 0；invalid 结果缺少 blocking issue 数为 0。
- [ ] required 路线/三餐/酒店/accepted constraint 缺失被误判 warning 或 complete 的次数为 0。
- [ ] opening hours 缺失被伪装成已验证或阻断的次数为 0；有来源冲突漏报数为 0。
- [ ] walking/transit 接驳超限被选择或未报告的次数为 0；mixed/self_drive 语义混淆数为 0。
- [ ] Validator 修改输入路线、时间、餐厅、酒店、活动顺序、预算或生成 TripPlan 的次数为 0。
- [ ] 默认测试外部调用数为 0，敏感数据日志/证据命中数为 0；Stage 1～12A 回归与前端构建通过。

## 11. 迁移与回滚

- Stage 12B 只新增内部验证结果，无 API/持久化/前端迁移。
- 回滚先撤销 Coordinator Validator 调用，再撤销 Validator、模型增量和测试；保留 Timeline/Budget 结果供后续重新接入。
- Stage 13A 以后依赖时按 13D→13C→13B→13A→12B 逆序回滚；不得以忽略问题或降低 severity 代替回滚。
- 回滚后执行 Stage 1～12A 测试、模式回归和前端构建，期望退出码为 0。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_12b/test-summary.md` | 提交标识、三条命令、17 用例、离线声明和失败数 |
| 规则矩阵 | `doc/opt_strategy/evidence/stage_12b/validation-rule-matrix.md` | 校验域、问题码、severity/scope 与测试 ID |
| 约束矩阵 | `doc/opt_strategy/evidence/stage_12b/constraint-trace-matrix.md` | accepted/ignored、required、饮食/移动/时间语义与测试 ID |
| 警告矩阵 | `doc/opt_strategy/evidence/stage_12b/warning-matrix.md` | unavailable 来源字段、提示和不阻断边界 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_12b/rollback-check.md` | 调用撤销、依赖逆序、命令和结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 11、12A 实际 Done，并在开工前复核 timeline/budget 模型、accepted constraint 语义、repair scope 与 Stage 13A/13B 的消费边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
