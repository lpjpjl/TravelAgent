# Stage 13B：路线、餐厅与酒店局部修复

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 13B |
| 目标 | 对不改变 PrimaryDraft 点位集合/日期/顺序的路线、餐厅、酒店问题进行有限、可追溯的局部修复，并重建所有受影响下游结果 |
| 对应优化项 | P0-7 行程深度关联化 |
| 前置依赖 | Stage 13A |
| 后续阻断 | 未通过时不得开始 Stage 13C、13D、14、15、16、18、20 |
| 默认流量影响 | 无；只扩展显式新链路内部 repair outcome，legacy/shadow 普通请求和 new reserved 不变 |

## 2. 当前代码基线与差异

- Stage 13A 对 Validator.invalid 只返回 requires_repair，不改变任何中间结果；当前不存在修复上下文、尝试上限、重复状态检测或问题优先级。
- Stage 10A/10B 有真实路线候选，Stage 9A 有同类型酒店/便餐检索与选择入口，Stage 11/12A/12B 可以重算和复验，但尚未形成局部替换闭环。
- 当前 legacy fallback 会用文本/默认值继续返回结果；目标修复只操作真实 source ID，不能用直线、默认北京坐标、LLM 事实或修改 accepted constraints 逃避问题。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/trip.py` | 增加 RepairContext、RepairAttempt、草案指纹、尝试结果和稳定停止原因 | 全局/同类计数、尝试 ID、已试 ID 不变量严格校验 |
| 新增 | `backend/app/services/local_repair_service.py` | 按优先级生成/执行 route、meal、lodging 局部替换方案及失效范围 | 不改 PrimaryDraft，不输出 TripPlan |
| 修改 | `backend/app/services/route_service.py` | 为指定 route requirement 重取/复用合格真实替代路线并排除已试 route ID | 不生成直线或跨约束回退 |
| 修改 | `backend/app/services/meal_search_service.py` | 为指定餐次按原锚点检索同级真实候选并重做饮食/营业预筛 | 不改其他餐次或主要点位 |
| 修改 | `backend/app/services/lodging_search_service.py` | 为指定 lodging boundary 检索同住宿类型真实候选 | 不改变住宿偏好或添加住宿晚数 |
| 修改 | `backend/app/services/lodging_meal_selection_service.py` | 从同级白名单选择替代酒店/便餐 ID 并保留 required/constraint 语义 | 无候选为业务不可行，不选第一条猜测 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 编排有限 repair loop、失效重建、Validator 重跑与 requires_repair 交接 | 顺序固定，不构造 partial/failed |
| 修改 | `backend/app/config.py`、`backend/.env.example` | 接入 `TRIP_REPAIR_MAX_ATTEMPTS` 与 `TRIP_REPAIR_MAX_SAME_TYPE_ATTEMPTS` 校验 | 默认 8/3；非法组合启动失败 |
| 新增 | `backend/tests/unit/services/test_local_repair_service.py` | 覆盖优先级、方案、评分、失效与循环停止 | 使用 fake，不访问公网 |
| 新增 | `backend/tests/unit/models/test_repair_context.py` | 覆盖计数、指纹、已试 ID、停止原因 | request-scoped 且稳定序列化 |
| 新增 | `backend/tests/integration/test_trip_coordinator_local_repair.py` | 覆盖路线/餐厅/酒店重建、上限与交接 | 无普通流量双跑 |

本 Stage 不重排 PrimaryDraft、修改 meal_role/day_part、删除点位、构造 partial/failed、映射 API 错误或变更前端。它不实现缓存或天气重建。

## 4. 输入、输出与错误契约

### 4.1 RepairContext 与优先级

仅当 Stage 13A 得到 `CoordinationOutcome.requires_repair` 且所有输入均为本次 request_id 的可信结构化对象时进入：

- `RepairContext` 包含 request_id、递增 attempt_id、global_attempts、按 `route|meal|lodging` 计数、已试 route/meal/hotel IDs、canonical draft fingerprint、问题摘要与停止原因；不得保存自由文本、完整 payload 或完整 Prompt；
- 全局最多 8 次、同类最多 3 次；由 Settings 校验 `0 <= same_type <= total`。指纹为 PrimaryDraft、LodgingMealPlan、ItineraryDraft、选择路线 ID 和受影响 ID 的 canonical JSON SHA-256 摘要；同一指纹再次出现立即停止 `duplicate_repair_state`；
- 一轮仅处理优先级最高的一个稳定问题：必要路线/方式 → 餐厅 → 酒店。accepted constraints、required ID、PrimaryDraft 日期/顺序/day_part/meal_role/removal_rank 在本 Stage 不可变；
- 计数达到上限、无候选或重复指纹时返回 requires_repair（含未关闭问题与停止原因），不生成 partial/failed；技术错误原样传播。

### 4.2 路线修复

仅处理 `route_mode_ineligible`、`missing_required_route` 或一次 transit 刷新后的路线问题：

- RouteService 对同一端点、城市、交通偏好、有效移动约束和查询参考取得真实候选，排除 RepairContext 已试 route ID；walking/transit 接驳约束、mixed 不含 self_drive 等 Stage 10 规则仍完整适用；
- 从剩余合格候选按 `(duration_seconds, distance_meters, route_id)` 确定性选择；替换 route ID 后只使该段、所属日 Timeline、预算、Validator 失效并重建；
- transit 实际时刻偏差仍沿用每段最多一次刷新；不得为“修复”第二次刷新、切直线或用未过滤候选。

### 4.3 餐厅修复

餐饮锚点、营业时间或餐窗问题只替换对应同日同 meal_role 的 convenience/gourmet 同级实体：

- 检索/筛选仍执行 DietaryRuleSet、真实来源、对应固定餐窗连续 >=40 分钟预筛；gourmet 仅能在已有可替换 gourmet 候选中替换，不能改变 food 下限、meal_role 或 required 约束；
- 为每个候选实际计算 `前锚点→餐厅→后锚点` 合格路线的总 `(duration_seconds, distance_meters, poi_id)`；末日晚餐只有前段，禁止直线或虚构后锚点；按该三元组选择，排除已试 meal ID；
- 替换后重建该餐次、完整 ItineraryDraft、受影响路线、Timeline、Budget、Validator；其他日期与主要点位不变。无合格候选保留问题并交后续 Stage。

### 4.4 酒店修复与失效范围

酒店导致必要路线缺失或时间轴不可行时，只替换相同 `AccommodationPreference` 的真实候选：

- 新酒店不得违反 accepted constraints 或改住宿晚数；通过 Stage 9A 同级选择入口并排除已试 hotel ID；
- 替换 Day i 的 lodging 后，必须重建 Day i-1 的 end boundary（如适用）、Day i start boundary、相邻餐次锚点/便餐、完整草案、所有受影响路线、Timeline、Budget 与 Validator；Day i 的 `start_hotel_id` 与前日 `end_hotel_id` 仍严格相等；
- 任何 rebuild 不得保留旧酒店相关 route/timeline/budget 事实，也不得改变 PrimaryDraft/accepted constraints。无候选为业务不可行 requires_repair。

### 4.5 输出、错误与日志

每轮输出 `RepairAttempt(status=repaired|no_candidate|limit_reached|duplicate_state)` 和新 ValidationResult：

- valid 时交 TripAssembler 形成 complete；invalid 时继续下一轮或向 Stage 13C 交接 requires_repair；
- Provider/LLM/schema/规则技术错误保持技术错误；合法无候选、重复/上限是业务 repair stop，不能伪装成技术错误或成功；
- 日志只记录 request_id、attempt、kind、问题码、候选/过滤数、已试 ID 摘要、fingerprint、status 和耗时；不得记录敏感事实正文。

## 5. 配置与功能开关

| Settings 字段 | 环境变量 | 默认值 | 校验 |
|---|---|---|---|
| `trip_repair_max_attempts` | `TRIP_REPAIR_MAX_ATTEMPTS` | 8 | 非负整数 |
| `trip_repair_max_same_type_attempts` | `TRIP_REPAIR_MAX_SAME_TYPE_ATTEMPTS` | 3 | 非负且不大于总上限 |

不新增用户流量开关；所有其他规则沿用前置 Stage。legacy/shadow 普通请求不进入 repair loop，new 在 Stage 16 前仍 reserved。

## 6. 实施步骤

1. 定义 RepairContext/Attempt/fingerprint/停止原因和 Settings 校验。
2. 实现问题稳定排序、上限/重复检测和 route 局部替换重建。
3. 实现真实餐厅两段路线评分、同级替换与局部下游重建。
4. 实现同类型酒店替换、相邻日期边界/餐次/路线/时间轴/预算全量失效重建。
5. 接入 Coordinator repair loop，valid→complete，未闭合问题→13C，技术错误原样抛出。
6. 执行 Stage 1～13B 回归与前端构建，归档 repair/循环/回滚证据。

## 7. 明确不做

- 不重排日期/顺序、改 day_part/meal_role、删除 optional 点或重建 PrimaryDraft。
- 不生成 partial/failed、HTTP 错误、前端状态或默认切换。
- 不用价格、天气、直线距离或 LLM 自由文本作为修复评分事实。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S13B-T01 | unit | 问题优先级 | route→meal→lodging，稳定同类排序 |
| S13B-T02 | unit | 3km walking/移动约束 | 换合格真实 taxi/transit，不判失败/不走直线 |
| S13B-T03 | integration | transit 刷新 | 最多一次刷新并重算 timeline |
| S13B-T04 | unit | 餐厅两段评分/末日晚餐 | 真实路线三元组选择；末日仅前段 |
| S13B-T05 | unit | 餐厅饮食/营业过滤 | 冲突/不合格候选不被选中 |
| S13B-T06 | integration | 替换餐厅 | 仅餐次及下游重建，主要点位不变 |
| S13B-T07 | integration | 替换酒店 | 相邻日边界、餐次、路线、timeline、budget 全失效重建 |
| S13B-T08 | unit | 同类型/accepted 约束 | 不换住宿类型、不修改 accepted/required |
| S13B-T09 | unit | 无候选 | requires_repair，无 partial/failed/fallback |
| S13B-T10 | unit | 8/3 上限 | 精确停止，不超额调用 |
| S13B-T11 | unit | 重复 fingerprint/已试 ID | 立即 duplicate_state，不循环 |
| S13B-T12 | unit | Provider/LLM 技术错误 | 原样传播，不转无候选 |
| S13B-T13 | integration | valid repair | Validator valid 后 assembler complete |
| S13B-T14 | integration | 未闭合交 13C | 保留 requires_repair/RepairContext，无重排 |
| S13B-T15 | regression | legacy/shadow/new 与上游 | 普通旧流量 repair 调用 0；回归不退化 |

```text
python -m pytest backend/tests/unit/models/test_repair_context.py backend/tests/unit/services/test_local_repair_service.py backend/tests/integration/test_trip_coordinator_local_repair.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，默认离线。Stage 13B 必测用例数为 15。

## 9. 人工验收

不适用：本 Stage 无普通 API/UI；repair 次数、路线评分、酒店失效和循环边界均由 FakeAmapProvider/FakeLLM/固定时钟自动覆盖。

## 10. 量化退出指标

- [ ] S13B-T01～T15 全通过，三条命令退出码 0。
- [ ] 真实 route/meal/hotel 修复后残留旧下游对象数为 0；PrimaryDraft/accepted 约束漂移数为 0。
- [ ] global>8、同类>3、重复 fingerprint 后继续 repair 的次数为 0。
- [ ] 直线/默认坐标/LLM 事实、partial/failed 在本 Stage 产生次数为 0。
- [ ] 默认外部调用数为 0，敏感日志/证据命中数为 0；Stage 1～13A 回归与前端构建通过。

## 11. 迁移与回滚

- Stage 13B 只增加显式新链路 repair loop，无数据/API/前端迁移。
- 回滚先移除 Coordinator loop 和 Settings 读取，再撤销 local repair/模型/测试；保留 13A happy path。
- 13C/13D 已依赖时按 13D→13C→13B 逆序回滚，不得以禁用 Validator 或恢复虚构 fallback 替代。
- 回滚后 Stage 1～13A、模式/API 回归与前端构建均须通过。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_13b/test-summary.md` | 提交、命令、15 用例、离线声明、失败数 |
| Repair 矩阵 | `doc/opt_strategy/evidence/stage_13b/repair-priority-matrix.md` | 问题→方案→失效范围→测试 ID |
| 餐厅/酒店矩阵 | `doc/opt_strategy/evidence/stage_13b/local-replacement-matrix.md` | 锚点路线、同类型、边界重建与测试 ID |
| 循环矩阵 | `doc/opt_strategy/evidence/stage_13b/repair-context-matrix.md` | 8/3、已试 ID、指纹、停止原因 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_13b/rollback-check.md` | loop 撤销、依赖逆序、命令、结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待第二批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 13A 实际 Done，并复核 Validator scope、RouteService、9A 搜索/选择与 13C 交接边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
