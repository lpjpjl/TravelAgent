# Stage 7B：DestinationResearchAgent 主要点位筛选

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 7B |
| 目标 | 让 DestinationResearchAgent 从 Stage 7A 的真实主要点位候选池中返回受严格校验的候选 ID 子集，同时保证 required 与 accepted 约束不被遗漏或伪造 |
| 对应优化项 | P0-4 结构化草案输出、P0-5 餐饮候选检索与筛选、P0-7 行程深度关联化 |
| 前置依赖 | Stage 5、7A |
| 后续阻断 | 未通过时不得开始 Stage 8、9A、13A、18 |
| 默认流量影响 | 仅扩展显式新链路/bootstrap；legacy 与普通 shadow 请求不调用主要点位筛选，默认 `/trip/plan` 响应不变 |

## 2. 当前代码基线与差异

- legacy `MultiAgentTripPlanner` 通过自然语言 Prompt 让 Agent 同时搜索、筛选并规划，无法证明输出点位来自本次真实候选集合。
- Stage 5 已建立 `DestinationResearchAgent` 与 `StructuredLLMAdapter`，但当前 Agent 只解析额外要求，不承担候选筛选；额外要求解析耗尽后转 ignored 的例外不能复用于核心筛选输出。
- Stage 7A 已交付 `PrimaryCandidatePool`：候选 ID、类型、偏好命中、required、constraint IDs、来源事实引用和确定性排除结果均已锁定。
- 当前没有主要点位筛选请求/响应 schema、候选 ID 白名单对账、required 完整性校验、ignored 约束隔离、筛选重试或耗尽错误。
- Stage 8 才负责分天、排序、`day_part`、`meal_role`、建议时长、`provisional_meal_slots` 和 `removal_rank`；本 Stage 输出不能包含这些字段。
- Stage 18 才让有效天气参与主要点位筛选/替换；本 Stage 初次交付时天气对 Prompt、选择和理由均为零影响。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 修改 | `backend/app/agents/destination_research_agent.py` | 增加主要点位筛选入口与固定 Prompt；只接收结构化候选上下文，不挂工具、不访问外部服务 |
| 修改 | `backend/app/models/research.py` | 增加 `PrimaryCandidateSelectionItem`、`PrimaryCandidateSelection`、`PrimarySelectionBusinessInfeasible` 判别联合与严格输出不变量 |
| 新增 | `backend/app/services/primary_candidate_selection_service.py` | 构建脱敏 Agent 输入、调用结构化适配器、对账 ID/类型/约束/required 并有限重试 |
| 新增 | `backend/app/services/primary_candidate_selection_errors.py` | 定义筛选耗尽等核心技术错误与稳定错误码 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 ready 候选池后串行调用筛选服务并写入 ResearchBootstrapResult |
| 修改 | `backend/tests/contract/test_destination_research_agent.py` | 增加筛选 JSON schema、固定温度、Prompt 边界、重试与耗尽测试 |
| 新增 | `backend/tests/unit/services/test_primary_candidate_selection_service.py` | ID 白名单、required、约束、重复、类型及空池分流测试 |
| 修改 | `backend/tests/integration/test_trip_coordinator_bootstrap.py` | 增加 Stage 7A→7B 接入、业务不可行短路、ignored/天气零影响和模式回归 |

本 Stage 不修改 AmapService、Provider、候选检索/构建策略、ItineraryPlannerAgent、API 路由、legacy Agent、前端类型或页面。

## 4. 输入、输出与错误契约

### 4.1 调用前置与输入快照

- 只允许 `PrimaryCandidatePool.status=ready` 进入筛选；`business_infeasible` 由 Coordinator 原样短路，不调用 LLM。
- ready 空池不调用 LLM，返回可信业务不可行 `PrimarySelectionStatus=business_infeasible`、`reason_code=primary_candidate_pool_empty`；不得让 Agent 虚构候选。
- 非空输入包含 request_id、city、travel_days、请求顺序的 preferences、normalized accepted constraints，以及按 7A 稳定顺序的候选视图。
- 候选视图只提供筛选所需字段：稳定 candidate ID（即 provider + source ID 的领域引用）、kind、具体名称、地址、来源 POI 类型、可用评分/营业信息及其 data quality、matched preferences、required、constraint IDs。缺失可选事实保持 null/unavailable。
- 不向 Prompt 提供 Provider 原始 payload、Key、自由文本、ignored 子句原文、酒店/便餐、路线、天气、预算或 legacy Agent 输出。
- accepted constraints 只提供 normalized 执行字段和稳定 ID；系统优先级明确为“系统真实性边界 → accepted 约束 → required →普通偏好/筛选建议”。Agent 不得改变、补充或删除约束。

### 4.2 Agent 调用规范

- 复用 Stage 5 `StructuredLLMAdapter`，使用 JSON mode/`response_format`、`temperature=0`，响应字符串必须直接 `json.loads` 后进入 `extra="forbid"` Pydantic 模型；禁止正则截取、Markdown 代码块修复或宽松字段丢弃。
- `DestinationResearchAgent.select_primary_candidates()` 只返回结构化选择，不调用 MCPTool/AmapService，不执行检索、事实补全、天气判断、分天或路线估算。
- Prompt 明确 Agent 可以根据真实名称、类型、位置描述、来源可选事实和 matched preferences 进行筛选推理，但只能操作输入 candidate ID；输出理由属于 generated 决策说明，不得伪装为 sourced 事实。
- 选择保持集合语义：输出顺序只用于稳定序列化与对账，不代表游览先后；Stage 8 必须独立生成日期和活动顺序。

### 4.3 输出模型

筛选服务返回以 `status` 为判别字段的 `PrimaryCandidateSelectionResult`：

- `status=selected` 时使用 `PrimaryCandidateSelection`；
- `status=business_infeasible` 时使用 `PrimarySelectionBusinessInfeasible`，只允许 request_id、稳定 reason_code 和输入计数，不包含 items 或伪造选择。

`PrimaryCandidateSelection` 至少包含：

- request_id、`status=selected`；
- `items`：非空且 candidate ID 唯一，每项只含 `candidate_id`、与候选池一致的 `kind`、原样 `constraint_ids` 和简短 `selection_reason`；
- `selected_count`、`attraction_count`、`gourmet_count`、`required_count`，由 items 与候选池事实交叉计算；
- `input_candidate_count` 与候选池 total_count 相等，便于审计但不允许 Agent任意填写。

`selection_reason` 限制为去除首尾空白后 1～200 个 Unicode code point；不得包含未提供的评分、票价、开放时间、榜单名次或天气结论。程序只校验长度和禁止明显的未知 ID/约束引用，事实真实性由 Agent 输入最小化、输出不承载事实字段和后续人工/契约样例共同约束。

输出不复制 POI 名称、地址、坐标或来源元数据；Coordinator 按 candidate ID 从 7A 池回填事实。输出不包含日期、顺序、`day_part`、`meal_role`、时长、`removal_rank`、酒店、便餐、路线、时间或预算。

### 4.4 确定性输出校验

每次 Agent 响应必须依次通过：

1. JSON 与 `extra="forbid"` schema 校验；
2. request_id、input count 和派生 count 对账；
3. 每个 candidate ID 存在于本次 7A 池，且无重复；
4. kind 与候选池完全一致，Agent 不得把 attraction 改为 gourmet 或反向修改；
5. item.constraint_ids 与候选池该实体的 accepted constraint IDs 完全一致，不得引用 ignored/未知 ID，也不得遗漏 required 关联约束；
6. 7A 池中全部 `required=true` 候选有且只有一次出现在 items；required 遗漏或降级为普通项均非法；
7. 非 required 候选可被舍弃，但不得输出输入池以外的替代实体；
8. 当 `food_preference_enabled=true` 且池中存在普通 gourmet 时，筛选结果至少包含 1 个 gourmet，为 Stage 8 的全程下限提供可规划输入；每天最多 1 个及 complete/partial 的 1～N/1～K 最终约束仍由 Stage 8 和后续完整性校验执行；
9. 当 food=false 但存在 required gourmet 时必须保留；不得根据普通偏好开关删除 required。

本 Stage 不规定景点总数或“每天 2～3 个”的最终安排数量。Agent 可在非 required 候选中筛除低相关项；Stage 8 根据日期、顺序和可执行性形成草案，候选不足由后续业务状态处理，不能在本 Stage 用任意固定数量冒充可执行性判断。

### 4.5 有限重试与耗尽

- 首次结构化输出失败后最多重试 2 次，总调用上限为 3；JSON/schema 非法、未知/重复 ID、kind 篡改、计数不符、constraint IDs 不符、required 遗漏或 food 下限不符均可重试。
- 每次重试使用完全相同的输入候选快照、temperature 和 schema，只追加稳定的脱敏校验摘要，例如 `unknown_candidate_id`、`required_candidate_missing`；不得回显完整 Prompt、POI 列表或用户原文。
- 任一合法响应立即停止。三次均失败时抛 `PrimaryCandidateSelectionError(error_code="primary_selection_retry_exhausted")`，作为核心规划链路技术错误；不得像 Stage 5 额外要求解析那样转 ignored，也不得返回部分选择、business_infeasible 或虚构 fallback。
- LLM 配置缺失、认证、限流、超时等技术异常由 StructuredLLMAdapter 保留稳定类别；是否允许重试遵循其核心调用策略，总尝试不得超过 3，耗尽后不改变为业务失败。

### 4.6 业务不可行、技术错误与日志

- 7A ready 空池、food 开启但 7A 没有任何 gourmet，属于可信业务不可行，调用 LLM 次数为 0；reason 分别为 `primary_candidate_pool_empty`、`gourmet_candidate_unavailable`。
- 只有普通 attraction 候选且 food=false 时可以筛选；只有 required 候选时仍调用 Agent并强制全部保留，使其输出结构化理由，但不得允许舍弃 required。
- Agent 合法选择后点位是否能覆盖全部日期、每天至少一个主要点、餐次和时间轴，留给 Stage 8 以后判定。
- 非法 Agent 输出/重试耗尽、LLM 技术异常、输入模型不一致属于技术错误；空候选或确定性缺少 gourmet 属于业务不可行，两类不得互换。
- 日志按 request_id 记录 stage=`primary_candidate_selection`、attempt、input/selected/required/gourmet count、status、稳定 validation/error code 和 duration；不得记录 Key、Prompt、自由文本、完整候选、完整 Agent 响应或 POI事实。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或功能开关；筛选温度 0、最多重试 2 次、理由长度和验证顺序是受测试锁定的代码策略。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求不运行 Coordinator；显式 shadow 验证可执行 Stage 7B；new 仍为 reserved。
- 复用现有 LLM 配置和 StructuredLLMAdapter 注入；DestinationResearchAgent 不读取 Settings、不持有 AmapService/MCPTool。

## 6. 实施步骤

1. 在 research 模型中定义 selected/business_infeasible 判别联合、selection item 的严格 schema、唯一性和派生计数正反例。
2. 为 DestinationResearchAgent 增加独立筛选 Prompt 与入口，固定 JSON mode、temperature=0 和只输出 ID/类型/约束/理由的边界。
3. 实现候选视图构建，确保只暴露来源事实、matched preferences 和 normalized accepted constraints，不携带 ignored、天气或自由文本。
4. 实现输出白名单对账、kind/constraint IDs 一致性、required 全保留和 food 最小 gourmet 输入校验。
5. 实现最多 2 次重试、稳定脱敏错误反馈及 `primary_selection_retry_exhausted` 核心技术错误。
6. 实现空池、gourmet 不可用的零 LLM 业务短路，以及 required-only 的强制保留路径。
7. 接入 TripCoordinator：7A ready 后调用筛选服务，将合法 selection 写入 ResearchBootstrapResult；保持天气仅展示、ignored 和模式分发行为不变。
8. 执行 Stage 1～7B 定向测试、后端全量回归和前端构建，归档 schema、ID 对账、重试、边界与回滚证据。

## 7. 明确不做

- 不搜索、补充、改写或重新排序事实候选，不改变 7A 的饮食过滤和 required 身份。
- 不生成日期、游览顺序、`day_part`、`meal_role`、建议时长、provisional meal slots 或 removal rank；属于 Stage 8。
- 不检索/筛选酒店和便餐，不校验营业餐窗，不生成路线、时间轴、预算或 TripPlan。
- 不根据天气筛选、替换或标记室内外；Stage 18 接入天气后必须重新评审本入口的输入与重建边界。
- 不建立榜单、排队时间、菜单级饮食安全或高级餐饮策略；属于 Future F-2。
- 不修改 API/前端/legacy 链路，不开放 new 模式，不删除 Stage 5 shadow 防护或 legacy fallback。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S7B-T01 | contract | 合法 attraction + gourmet 选择 | JSON mode、temperature=0、extra=forbid，输出只含允许字段 |
| S7B-T02 | contract | 输出额外事实/日期/顺序字段 | schema 拒绝并重试 |
| S7B-T03 | unit | candidate ID 白名单 | 所有选择均来自本次 7A 池，未知 ID 被拒绝 |
| S7B-T04 | unit | 重复 candidate ID | 拒绝并报告 duplicate_candidate_id |
| S7B-T05 | unit | kind 被修改 | 拒绝并报告 candidate_kind_mismatch |
| S7B-T06 | unit | required 候选遗漏 | 拒绝并报告 required_candidate_missing |
| S7B-T07 | unit | required 与普通候选重合 | 输出仅一次且 required/constraint 对账通过 |
| S7B-T08 | unit | constraint IDs 遗漏、未知或 ignored 引用 | 均拒绝，不让 ignored 影响筛选 |
| S7B-T09 | unit | 非 required 候选被舍弃 | 合法，输入事实不被改写 |
| S7B-T10 | unit | food=true 且有 gourmet | 合法结果至少选择 1 个 gourmet |
| S7B-T11 | unit | food=true 但 Agent 未选 gourmet | 非法并重试，不提前校验每天最多 1 个 |
| S7B-T12 | unit | food=false + required gourmet | required gourmet 必须保留，不受普通开关影响 |
| S7B-T13 | unit | ready 空池 | 零 LLM，business_infeasible/primary_candidate_pool_empty |
| S7B-T14 | unit | food=true 但池中无 gourmet | 零 LLM，business_infeasible/gourmet_candidate_unavailable |
| S7B-T15 | unit | required-only 候选池 | 调用 1 次成功并全部保留，允许 matched preferences 为空 |
| S7B-T16 | contract | 首次成功、第三次成功 | 调用数分别为 1、3，重试输入快照不变 |
| S7B-T17 | contract | 三次 schema/ID 校验失败 | 抛 primary_selection_retry_exhausted，不返回部分选择 |
| S7B-T18 | contract | LLM 超时/认证等技术异常耗尽 | 保持技术错误类别，不转换业务失败或 ignored |
| S7B-T19 | unit | selection_reason 空、超 200 字 | schema 拒绝；合法理由标记为 generated 决策说明 |
| S7B-T20 | unit | input/selected 派生计数不一致 | 严格对账拒绝，不信任 Agent 计数 |
| S7B-T21 | contract | Agent 依赖边界 | 无 MCPTool/AmapService；不导入 legacy planner或 ItineraryPlannerAgent |
| S7B-T22 | integration | 7A ready → 7B selected | Coordinator 保存 selection，按 ID 可回填唯一事实 |
| S7B-T23 | integration | 7A business_infeasible | 不调用筛选 Agent，状态与 reason 原样短路 |
| S7B-T24 | integration | ignored 和天气变化 | 两次输入/选择一致；Stage 18 前二者对筛选零影响 |
| S7B-T25 | regression | legacy/shadow/new 模式 | 普通 legacy/shadow 筛选调用数 0，new 仍 planner_mode_not_ready |
| S7B-T26 | regression | Stage 1～7A 与前端 | Provider、解析、约束、检索、候选池、shadow 测试和构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_primary_candidate_selection_service.py backend/tests/contract/test_destination_research_agent.py backend/tests/integration/test_trip_coordinator_bootstrap.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实 LLM。Stage 7B 必测用例数为 26，失败数必须为 0。

## 9. 人工验收

不适用：Stage 7B 不接入普通 API/UI；候选白名单、required、约束、重试与技术/业务分流均由 FakeLLM 和固定候选 fixture 自动化覆盖。显式 shadow 人工入口沿用 Stage 5，不重复建立页面验收。

## 10. 量化退出指标

- [ ] S7B-T01～S7B-T26 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] Agent 输出未知或重复 candidate ID 数为 0；kind、constraint IDs 和派生计数不一致的输出通过数为 0。
- [ ] 7A 全部 required 候选保留率为 100%，每个 required 在 selection 中恰好出现 1 次，遗漏数为 0。
- [ ] ignored constraint 引用进入 Prompt/selection 的数量为 0；自由文本、天气、酒店、便餐和路线字段进入筛选输入的数量为 0。
- [ ] food=true 且池中存在 gourmet 时合法 selection 的 gourmet_count ≥1；每天最多 1 个与最终 1～N/1～K 未在本 Stage 越界判定。
- [ ] 空池和 gourmet 不可用路径 LLM 调用数均为 0，并返回稳定 business_infeasible；required-only 合法路径全部 required 保留。
- [ ] 合法输出不复制或改写 POI 名称、地址、坐标、评分、营业时间；事实回填只按本次 candidate ID 完成。
- [ ] 一次成功调用数为 1、第三次成功为 3、耗尽为 3；重试上限外调用数为 0。
- [ ] 核心筛选重试耗尽全部返回 primary_selection_retry_exhausted 技术错误，转 ignored、partial selection 或虚构 fallback 的数量为 0。
- [ ] Stage 18 前天气输入变化导致选择变化的用例数为 0；输出日期/顺序/meal_role/removal_rank 等越界字段数为 0。
- [ ] legacy/shadow 普通请求行为不变，new 仍 reserved；Stage 1～7A 回归与前端构建通过。
- [ ] 03 Stage 7B 每条退出条件及 02 的 ID 白名单、required、accepted 优先级均映射到测试/证据 ID，新增未追踪 TODO 数为 0。

## 11. 迁移与回滚

- Stage 7B 只在显式新链路的内部 ResearchBootstrapResult 增加主要点位 selection；默认用户流量仍为 legacy，无持久化数据迁移。
- 回滚时移除 Coordinator 的 selection 调用、筛选服务/错误和新增模型，恢复 Stage 7A PrimaryCandidatePool 作为 bootstrap 末端；Stage 1～7A 行为和数据保留。
- 回滚不得删除 DestinationResearchAgent 已有额外要求解析入口或 StructuredLLMAdapter；只撤销本 Stage 新增的筛选职责。
- 回滚后 Stage 1～7A 全量测试、legacy/shadow 模式测试和前端生产构建必须通过。
- Stage 8 以后已依赖 PrimaryCandidateSelection 时必须按依赖逆序回滚，不能单独移除筛选输出。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_7b/test-summary.md` | 提交标识、三条命令、26 个用例结果、离线声明和失败数 |
| 输出契约矩阵 | `doc/opt_strategy/evidence/stage_7b/selection-contract-matrix.md` | schema、ID/kind/constraint 对账、派生计数与测试 ID |
| required 与偏好矩阵 | `doc/opt_strategy/evidence/stage_7b/required-preference-matrix.md` | required 全保留、food 开关、gourmet 下限、空池分流 |
| 重试与错误矩阵 | `doc/opt_strategy/evidence/stage_7b/retry-error-matrix.md` | 1/3/耗尽调用、脱敏反馈、技术/业务分流和错误码 |
| 边界审计 | `doc/opt_strategy/evidence/stage_7b/stage-boundary-check.md` | 7A 输入、7B 输出及未越界进入 Stage 8/9A/18 的检查结果 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_7b/rollback-check.md` | Coordinator 恢复点、Agent 入口撤销、依赖逆序、命令与结果 |

## 13. Ready/Done 复核

当前设计状态：**工作包已细化，R2～R10 设计复核通过；执行 Ready 待 R1**。R1 必须等待 Stage 5、7A 实际 Done；第二次跨 Stage 审计已确认 7A `PrimaryCandidatePool` 到本 Stage `PrimaryCandidateSelection` 的 ID/required/constraint 一致性、核心筛选耗尽不得复用 parse_failed 例外，以及 Stage 8 对日期、顺序和美食最终数量约束的唯一职责。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
