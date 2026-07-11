# Stage 13D：全局跨日重排与 partial/failed

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 13D |
| 目标 | 在局部修复/删点仍不可行时完成最后一次全日期跨日重排，并严格重建最长连续完整前缀为 partial 或返回 failed |
| 对应优化项 | P0-7 行程深度关联化、P1-1 结果状态基础 |
| 前置依赖 | Stage 13C |
| 后续阻断 | 未通过时不得开始 Stage 14、15、16、18、20、28 |
| 默认流量影响 | 无；只扩展显式新链路内部最终 outcome，默认 API/前端切换仍留 Stage 16 |

## 2. 当前代码基线与差异

- 13A 只能 complete，13B/13C 未闭合时保留 requires_repair；当前没有跨日候选重排、prefix 重建、partial/failed 模型或最长连续前缀保证。
- 直接截取旧 days 会把 DayK 错当末日、遗漏 `lodging[K]`、保留无效路线/预算或跳过失败日，违反 02 的 partial 规则。
- 当前 legacy fallback 可能伪造完整日；目标只从重新验证通过的真实事实构造 complete/partial，accepted constraint 无法满足时必须 failed。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/trip.py` | 收口 complete/partial/failed、missing items、首个失败日、degraded 与失败摘要模型 | partial/failed 语义互斥，failed 不伪装完整 days |
| 修改 | `backend/app/models/itinerary.py` | 支持显式 `planning_days` 的 partial lodging/meal/day-boundary 契约 | K<N 时 lodging 长度 K+1、DayK 非末日边界严格校验 |
| 新增 | `backend/app/services/partial_rebuild_service.py` | 执行全日跨日重排、K 前缀候选/住宿/餐次重建与 Validator 闭环 | 禁止旧数组切片和后续日期跳跃 |
| 修改 | `backend/app/services/primary_draft_service.py` | 支持受限全日跨日重排及 prefix 规划日期输入 | 不改变 accepted/required/事实边界 |
| 修改 | `backend/app/services/lodging_meal_selection_service.py` | 支持 K 日前缀的 K+1 lodging 和三餐完整性重建 | DayK 抵达 lodging[K]，不新增末日晚住宿 |
| 修改 | `backend/app/services/trip_assembler.py` | 在 valid complete/partial 输入上组装对应 TripPlan；构建失败摘要 | 不决定可行性或生成事实 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 13C 未闭合后执行全局重排→递减 K 重建→complete/partial/failed | 同一 RepairContext、无 API 切换 |
| 新增 | `backend/tests/unit/services/test_partial_rebuild_service.py` | 覆盖跨日、递减 K、住宿/food/constraint/失败边界 | Fake Provider/LLM，离线 |
| 修改 | `backend/tests/unit/models/test_budget_trip_error.py` | 覆盖 partial/failed/missing items/degraded 组合 | 连续前缀与 K+1 budget 语义严格 |
| 新增 | `backend/tests/integration/test_trip_coordinator_partial.py` | 覆盖 Day2/3 失败、complete/partial/failed 与无跳日 | 不访问公网 |

本 Stage 不实现 API 分级错误、默认 `/trip/plan` 切换、前端展示、缓存 degraded 来源或对话；分别由 14～17、31 以后 Stage 负责。

## 4. 输入、输出与错误契约

### 4.1 全日期跨日重排

仅当 13C 终止且仍 requires_repair 时，Coordinator 在完整 N 日范围先执行一次 `global_replan`：

- 基于原 PrimaryCandidateSelection 与 accepted constraints，PrimaryDraftService 可跨日调整 optional 主要点、美食级餐点和餐次位置；required ID、constraint IDs、事实、饮食规则、旅行日期范围和每日至多一个 gourmet 不可变；
- 全日重排后必须从 PrimaryDraft 依次重建 9A、9B、10A/10B、11、12A、12B；任何成功即由 assembler 输出 complete；
- 全局重排仍计入同一 RepairContext，受 8/3/duplicate fingerprint 约束；核心 Agent 重试耗尽保持技术错误，不转 partial；
- accepted explicit-time 目标无法在全日重排中满足，或 required/饮食/移动硬约束无法满足，直接进入 failed，不允许削减日期绕过。

### 4.2 连续 partial 前缀重建

仅在完整 N 日已完成全局重排仍不可行、且不存在上述 accepted constraint failed 条件时，按 `K=N-1, N-2, …, 1` 递减尝试：

1. 每个 K 都以 Day1..DayK 的 planning_dates、可保留/需重排的 selection 和约束重新调用 PrimaryDraft，而不是裁剪旧 `days`；不得返回 DayK 之后的任何日期。
2. 重建前缀的住宿必须为 `lodging[0..K]` 共 K+1 条：Day i 从 lodging[i] 出发；i<K 的 day end 到 lodging[i+1]；DayK 仍为原请求非末日，必须抵达 lodging[K]。预算按 K+1 晚重新计算。
3. 重建三餐、完整活动、真实路线、Timeline、Budget、Validator；每个 K 必须通过全部 Validator 才能生成 partial。失败则继续 K-1；K=0 返回 failed。
4. food=true 时 partial K 日必须含 1～K 个 gourmet、每天最多一个；必要时先在前缀内重排美食级餐点和餐次，仍无法满足即 failed。
5. 不得直接截取数组、跳过失败日返回后续日、用末日 17:00—18:00 规则处理 DayK，或因末日晚餐结束额外加住宿。

### 4.3 结果状态与失败摘要

| 状态 | 允许内容 | 不允许内容 |
|---|---|---|
| `complete` | N 个连续 valid days、N lodging、完整约束/路线/timeline/budget | missing required item、未闭合 Validator issue |
| `partial` | Day1..DayK（1≤K<N）连续 valid days、K+1 lodging、首个失败日/截断说明、missing optional/unavailable 提示 | 跳日、DayK 末日语义、未验证日、accepted constraint 缺失 |
| `failed` | request_id、稳定原因码、脱敏失败摘要、完整 accepted constraints | days、虚构事实、partial 数据伪装成功 |

- `degraded` 与 plan_status 正交，本 Stage固定 `degraded=false`，除非未来 Stage 17 输入经允许 stale cache；
- failed 原因仅用稳定 code，例如 `accepted_constraint_unsatisfied`、`no_feasible_complete_or_prefix`、`food_requirement_unsatisfied`；不得暴露内部 repair/LLM stack；
- TripAssembler 只能组装 Validator.valid 的 complete/partial；可行性判断、K 选择和 failed 原因仅由 Coordinator/PartialRebuildService 负责。

### 4.4 错误、日志与停止

- Provider/LLM/schema/配置错误在任意重建中原样抛技术错误，不生成 failed TripPlan 掩盖故障；
- 合法空候选、完整/前缀不可行属于业务结果，按上述 complete/partial/failed 分流；
- 日志只记录 request_id、attempt、global/prefix K、状态、首个失败日、计数、稳定 code、fingerprint 和耗时；完整 diagnostics 仅内存/脱敏证据，不能进 API/前端。

## 5. 配置与功能开关

不新增 Settings/开关；沿用 RepairContext 8/3 上限、Stage 2 结果状态模型与所有领域常量。legacy/shadow 普通请求不执行，new 在 Stage 16 前仍 reserved。

## 6. 实施步骤

1. 收口 TripPlan complete/partial/failed、missing items、first_failed_date 与 failed summary 模型。
2. 实现一次全日期跨日重排及全下游 rebuild，锁定 accepted/required/food 边界。
3. 实现递减 K prefix planner、K+1 lodging/预算、DayK 非末日路线/时间轴与全 Validator 闭环。
4. 实现 complete/partial assembler 输入和 failed summary；禁止旧数组切片/跳日。
5. 接入 Coordinator 最终有限状态机与 RepairContext；技术错误和业务结果严格分流。
6. 执行 Stage 1～13D 回归与前端构建，归档 prefix/状态/回滚证据。

## 7. 明确不做

- 不放宽 accepted constraints、删除 required、凭空增加事实或使用 LLM/默认坐标补齐。
- 不映射 HTTP、切默认 API、展示 partial/failed、实现 Pinia/时间轴 UI。
- 不实现 cache degraded、天气筛选、地图/对话/持久化或再次局部修复策略。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S13D-T01 | unit | 全日 cross-day replan 成功 | 重建全下游并 complete，required/accepted 不漂移 |
| S13D-T02 | unit | Day2 失败 N=3 | 先尝试 K=2，不切旧数组；Day1..2 valid 才 partial |
| S13D-T03 | unit | Day3 失败 N=3 | K=2 重建、Day2 非末日抵达 lodging[2]、预算 3 晚 |
| S13D-T04 | unit | K 递减连续失败 | K 依次递减，无跳日，K=0 failed |
| S13D-T05 | unit | partial DayK 末日规则 | 禁止 17—18 晚餐/无结束酒店语义，必须非末日边界 |
| S13D-T06 | unit | K+1 lodging/预算 | lodging[0..K]、房晚、单位与 total 正确 |
| S13D-T07 | unit | food=true 前缀 | 1～K gourmet、每日≤1；不满足则 failed |
| S13D-T08 | unit | accepted required/explicit/dietary/mobility 失败 | 直接 failed，不尝试 partial 绕过 |
| S13D-T09 | unit | failed shape/degraded | 无 days/虚构事实；degraded=false 与状态正交 |
| S13D-T10 | unit | complete/partial assembler 输入 | 仅 valid 输入可组装，unknown/issue 拒绝 |
| S13D-T11 | integration | 13C 未闭合→global→partial | 顺序、RepairContext、重建调用数稳定 |
| S13D-T12 | integration | 技术错误 | 原样错误，不构造 failed TripPlan |
| S13D-T13 | integration | first day 不可行 | K=0 failed，无 partial |
| S13D-T14 | regression | legacy/shadow/new 与上游 | 普通旧流量调用 0；Stage 1～13C/前端不回退 |

```text
python -m pytest backend/tests/unit/models/test_budget_trip_error.py backend/tests/unit/services/test_partial_rebuild_service.py backend/tests/integration/test_trip_coordinator_partial.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，默认离线。Stage 13D 必测用例数为 14。

## 9. 人工验收

不适用：本 Stage 不切 API/UI；全日/前缀重建、状态和预算均由 fixed fixtures/Fake Provider/LLM 自动覆盖。Stage 14/15/16 再验收呈现、错误和默认入口。

## 10. 量化退出指标

- [ ] S13D-T01～T14 全通过，三条命令退出码 0。
- [ ] direct slice、跳日、DayK 末日语义、K+1 lodging/预算错误的接受数均为 0。
- [ ] accepted constraint 不满足却生成 partial 的次数为 0；food 下限不满足却生成 partial 的次数为 0。
- [ ] complete/partial 含 invalid issue/未知事实/虚构 fallback 的次数为 0；技术错误伪装 failed 的次数为 0。
- [ ] degraded 被本 Stage 擅自置 true 的次数为 0；RepairContext 上限/重复状态绕过数为 0。
- [ ] 默认外部调用数为 0，敏感日志/证据命中数为 0；Stage 1～13C 回归和前端构建通过。

## 11. 迁移与回滚

- Stage 13D 仅增加显式新链路最终状态机，无默认 API/前端/持久化迁移。
- 回滚先移除 global/prefix 分支及 partial assembler，再撤销 PartialRebuildService/模型/测试，恢复 13C requires_repair；保留 13A complete happy path。
- 14/15/16 已依赖时按其逆序先回滚；不得通过把 partial 伪装 complete、恢复 legacy fallback 或丢弃失败日来替代。
- 回滚后 Stage 1～13C、模式/API 回归和前端构建均须通过。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_13d/test-summary.md` | 提交、命令、14 用例、离线声明、失败数 |
| 全局重排矩阵 | `doc/opt_strategy/evidence/stage_13d/global-replan-matrix.md` | 全日重排、不可变量、重建阶段、测试 ID |
| Prefix 矩阵 | `doc/opt_strategy/evidence/stage_13d/partial-prefix-matrix.md` | N/K、DayK、lodging、预算、food、测试 ID |
| 状态矩阵 | `doc/opt_strategy/evidence/stage_13d/plan-status-matrix.md` | complete/partial/failed/degraded/原因码与测试 ID |
| 回滚验证 | `doc/opt_strategy/evidence/stage_13d/rollback-check.md` | 分支撤销、依赖逆序、命令、结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待第二批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 13C 实际 Done，并复核全日/prefix rebuild、lodging/预算、accepted/food 状态边界和 Stage 14～16 的消费契约。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
