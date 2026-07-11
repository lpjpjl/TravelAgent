# Stage 13C：明确时间重排与删点重建

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 13C |
| 目标 | 对 13B 未闭合的明确时间约束冲突重建 PrimaryDraft；对普通超窗最后按 removal_rank 删除一个 optional 主要点，并从草案起完整重建受影响下游 |
| 对应优化项 | P0-7 行程深度关联化 |
| 前置依赖 | Stage 13B |
| 后续阻断 | 未通过时不得开始 Stage 13D、14、15、16、18、20 |
| 默认流量影响 | 无；仅显式新链路内部重建，legacy/shadow 普通请求和 new reserved 不变 |

## 2. 当前代码基线与差异

- 13B 仅能替换路线/餐厅/酒店，明确时间冲突或可选点过多时会保留 requires_repair；当前无 PrimaryDraft 重建、删除集合或下游失效图。
- Stage 8 可根据既有选择生成严格 PrimaryDraft，Stage 9A～12B 可重建住宿、餐次、完整草案、路线、时间轴、预算和校验；但尚未规定何时允许改变日期/顺序或删除 optional。
- legacy 可能压缩时长、跳过餐次或丢掉 required 点；目标流程禁止这些快捷修复。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/itinerary.py` | 增加 RebuildRequest、不可删除集合、删除清单与重排原因契约 | required/未知/重复删除 ID 拒绝 |
| 修改 | `backend/app/services/primary_draft_service.py` | 支持 explicit-time 重排和给定 optional 删除集的严格重建输入 | 只引用原 selection，保留 accepted 语义 |
| 修改 | `backend/app/agents/itinerary_planner_agent.py` | 增加受限 replan Prompt：只调整日期/顺序/day_part/meal_role，不生成事实 | JSON mode/temperature=0.3、ID 白名单 |
| 新增 | `backend/app/services/rebuild_service.py` | 计算删除候选、失效范围并从 PrimaryDraft 串行重建 9A～12B | 不直接改中间对象或跳过 Stage |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 13B 停止后按问题类别进入 explicit replan 或 optional deletion | 继承同一 RepairContext，不生成 partial/failed |
| 修改 | `backend/tests/unit/services/test_primary_draft_service.py` | 补 explicit-time 重排/删除输入约束 | required/constraint/food 规则不漂移 |
| 新增 | `backend/tests/unit/services/test_rebuild_service.py` | 覆盖失效图、rank、重建顺序与停止 | 离线 fake 调用序列可断言 |
| 新增 | `backend/tests/integration/test_trip_coordinator_rebuild.py` | 覆盖 13B→13C、重排/删点、未闭合交接 | 不访问公网/真实 LLM |

本 Stage 不替换路线/餐厅/酒店的局部事实策略，不做跨日全局优化或 partial/failed，不修改前端/API/默认模式。

## 4. 输入、输出与错误契约

### 4.1 进入条件与不可变量

仅接收 13B 停止且仍 invalid 的同 request_id `CoordinationOutcome.requires_repair` 与 RepairContext：

- accepted constraints、PrimaryCandidateSelection、required IDs、来源事实、交通偏好和天气零影响边界不可变；ignored 不得重新进入；
- 继续使用同一 RepairContext 的 global/同类计数、已试集合与 fingerprint，不能因重建重置 8/3 上限或绕过 duplicate state；
- `RebuildRequest` 只允许 `reason=explicit_time_replan|optional_removal`，并列出受影响日期、保留 selection IDs、删除 optional IDs 和稳定 issue references；技术错误原样传播。

### 4.2 明确时间约束重排

仅当 Validator 给出 `explicit_time_violation` 或 accepted explicit-time 目标未满足时：

- PrimaryDraftService 用原始完整 selection 和 normalized accepted constraints 调用受限 replan；Agent 只能调整原 candidate 的日期、日内顺序、attraction day_part 或 gourmet meal_role，必须保留全部 ID、kind、required、constraint IDs、建议时长与 removal_rank；
- 指定日期/时间目标关联 item 必须进入指定日期；具体可达性仍由重建后的 Route/Timeline/Validator 判定；
- 重排后从 9A 开始依次重建住宿/早餐/便餐、9B 完整草案、10A/10B 路线、11 时间轴、12A 预算、12B 校验；不得复用任何依赖旧日期/顺序/meal_role 的酒店、餐次、路线、timeline 或 budget；
- replan 重试耗尽保持 `primary_draft_retry_exhausted` 技术错误，不转删除/partial/fallback。

### 4.3 普通超窗与 optional 删除

只有不存在未满足 accepted explicit-time 约束，且 13B/重排仍报告普通 timeline 超窗时，才可删除 optional：

- 从 Validator 的稳定 optional repair candidates 中选取最小 `removal_rank`、未在 RepairContext 删除过的一个主要点；rank 相同或 required/酒店/三餐/accepted target/便餐/早餐均为契约错误；
- 删除美食级点时必须在重建结果中仍满足 food=true 的 1～N（或 partial 前缀未来规则）及每日最多一个 gourmet；对应餐次必须由 Stage 9A 重新补齐，不能删除后留下空餐次；
- 删除后以余下 selection 从 Stage 8 重新生成 PrimaryDraft，再完整串行重建 9A～12B；不直接从旧 ItineraryDraft 删除活动、不压缩路线/餐时/buffer；
- 选择集小于 travel_days、删除会移除 required、或无可删除 optional 时保留 requires_repair 交 13D，不把日期直接截断。

### 4.4 输出与日志

- 每轮记录 `RepairAttempt(kind=primary_replan|optional_removal)`、旧/新 fingerprint、影响日、删除 ID（如有）、重建阶段、Validator 结果和停止原因；
- valid 进入 TripAssembler complete；未闭合问题进入 13D requires_repair；本 Stage不产生 partial/failed；
- 日志仅含 request_id、attempt、reason、计数、rank、受影响日、指纹摘要、status/error code；不得记录 Prompt、用户文本、完整事实或 Agent 输出。

## 5. 配置与功能开关

不新增 Settings/开关；沿用 13B `TRIP_REPAIR_MAX_*`、Stage 8 temperature/重试及所有既有领域常量。普通 legacy/shadow 不执行，new 在 Stage 16 前 reserved。

## 6. 实施步骤

1. 定义 RebuildRequest、删除集、不可删除集合和重排原因模型。
2. 扩展 PrimaryDraft replan 输入/Prompt/校验，锁定 explicit-time 允许字段和 retry 语义。
3. 实现 deterministic optional removal 选择、food/餐次保护与失效图。
4. 实现从 Stage 8 或 9A 起的完整串行 rebuild，禁止旧下游对象复用。
5. 接入 Coordinator 与 RepairContext，valid 完成、未闭合交 13D。
6. 执行 Stage 1～13C 回归与前端构建，归档重排/删除/回滚证据。

## 7. 明确不做

- 不更改 accepted constraints、required、来源事实、交通偏好或候选池；不通过新增景点修复。
- 不作跨日全局可选点/美食重排或 partial/failed；属于 13D。
- 不缩短真实路线、40 分钟用餐或 30 分钟 buffer；不以旧日期数据局部拼接。
- 不映射 API/前端错误或切默认链路。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S13C-T01 | unit | explicit_time_violation | 仅进入 replan，不先删除 optional |
| S13C-T02 | contract | replan 输出 ID/required/constraint/duration/rank | 任一漂移拒绝并重试 |
| S13C-T03 | unit | 日期/顺序/day_part/meal_role 合法调整 | 指定日目标保留，其他事实不变 |
| S13C-T04 | contract | 三次 replan 非法/异常 | primary_draft_retry_exhausted 技术错误 |
| S13C-T05 | unit | 普通超窗 removal rank | 仅最小未试 optional 被删除 |
| S13C-T06 | unit | required/酒店/三餐/accepted target 删除 | 全部拒绝 |
| S13C-T07 | unit | 删除 gourmet | food 下限/每日上限/餐次补齐均重建 |
| S13C-T08 | unit | 无 optional/selection<days | requires_repair 交 13D，无数组截断 |
| S13C-T09 | integration | replan rebuild | 9A→12B 全量重建，无旧下游对象 |
| S13C-T10 | integration | optional deletion rebuild | Stage 8→12B 全量重建，路线/餐时/buffer 不压缩 |
| S13C-T11 | unit | RepairContext 继承/重复状态 | 不重置 8/3，不绕过指纹停止 |
| S13C-T12 | integration | valid/未闭合 | 前者 complete，后者仅交 13D |
| S13C-T13 | regression | legacy/shadow/new 与上游 | 普通旧流量调用 0；回归不退化 |

```text
python -m pytest backend/tests/unit/services/test_primary_draft_service.py backend/tests/unit/services/test_rebuild_service.py backend/tests/integration/test_trip_coordinator_rebuild.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，默认离线。Stage 13C 必测用例数为 13。

## 9. 人工验收

不适用：本 Stage 无页面/API 接入，重建范围、删除保护、重试和 RepairContext 由 FakeLLM/FakeProvider 自动覆盖。

## 10. 量化退出指标

- [ ] S13C-T01～T13 全通过，三条命令退出码 0。
- [ ] explicit-time 问题先进入 replan、普通超窗先于删除的次数为 0。
- [ ] required/accepted/三餐/酒店删除数为 0；删除后未重建 9A～12B 的次数为 0。
- [ ] 旧路线/timeline/budget复用、压缩餐时/buffer或生成 partial/failed 的次数为 0。
- [ ] RepairContext 上限/指纹绕过数为 0；Stage 1～13B 回归和前端构建通过。

## 11. 迁移与回滚

- Stage 13C 仅增加内部 replan/rebuild 分支，无数据/API/前端迁移。
- 回滚先撤销 Coordinator replan/delete 分支，再移除 RebuildService/模型/Agent 增量；恢复 13B requires_repair 交接。
- 13D 已依赖时先回滚 13D；不得以跳过重建、删除 required 或虚构 fallback 代替。
- 回滚后 Stage 1～13B、模式/API 回归和前端构建均须通过。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_13c/test-summary.md` | 提交、命令、13 用例、离线声明、失败数 |
| 重排矩阵 | `doc/opt_strategy/evidence/stage_13c/explicit-time-replan-matrix.md` | 原/新日期顺序、不可变量、测试 ID |
| 删除矩阵 | `doc/opt_strategy/evidence/stage_13c/optional-removal-matrix.md` | rank、保护项、food/餐次重建、测试 ID |
| 失效矩阵 | `doc/opt_strategy/evidence/stage_13c/rebuild-invalidation-matrix.md` | 起点、下游阶段、旧对象禁止复用 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_13c/rollback-check.md` | 分支撤销、依赖逆序、命令、结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待第二批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 13B 实际 Done，并复核 PrimaryDraft/9A～12B rebuild、RepairContext、food 规则和 13D 交接边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
