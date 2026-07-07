# Stage 7A：主要点位候选池

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 7A |
| 目标 | 将 Stage 6 的真实偏好命中与 Stage 5 的 required POI 确定性合并为第一选址优先级的主要点位候选池，并在入池前执行美食开关和饮食排除规则 |
| 对应优化项 | P0-5 餐饮候选检索与筛选、P0-7 行程深度关联化 |
| 前置依赖 | Stage 5、6 |
| 后续阻断 | 未通过时不得开始 Stage 7B、8、9A、13A、18 |
| 默认流量影响 | 仅扩展显式新链路/bootstrap；legacy 与普通 shadow 请求不构建候选池，默认 `/trip/plan` 响应不变 |

## 2. 当前代码基线与差异

- 当前 legacy Agent 直接在 Prompt 中请求景点、酒店和餐饮，没有正式 `CandidatePool`，也没有候选类型、required 身份或约束引用的程序化边界。
- Stage 2 已定义 `Candidate` 只引用真实事实 ID；Stage 5 已唯一绑定 accepted required POI 并建立确定性饮食规则；Stage 6 已产出 `PreferenceSearchResult`，保留真实 `SourcedPOI`、命中偏好与独立 required 引用。
- Stage 6 的 restaurant hit 只是偏好检索命中，尚未成为美食级主要点；无 `food` 偏好时不得因普通搜索结果类型自动提升餐厅。accepted required 餐厅遵循更高约束优先级，可独立成为美食级候选。
- 当前尚无主要点位候选池的稳定顺序、重复实体合并、候选分类、required 与 constraint IDs 合并、饮食过滤记录或候选充足性语义。
- Stage 7B 才调用 DestinationResearchAgent 做主要点位筛选；Stage 8 才生成日期、活动顺序、`day_part`、`meal_role` 和删除排名。上述职责不得提前进入本 Stage。
- 酒店和便餐依赖主要点位的空间锚点与顺序，必须留到 Stage 9A；本 Stage 不做相关检索，也不以占位实体伪造候选。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/services/primary_candidate_pool_service.py` | 合并偏好命中与 required POI、分类、去重、执行美食开关和饮食过滤，产出稳定候选池 |
| 新增 | `backend/app/services/candidate_pool_errors.py` | 定义候选事实冲突等技术错误；候选为空不使用技术错误 |
| 修改 | `backend/app/models/research.py` | 增加 `PrimaryCandidateKind`、`PrimaryCandidate`、`CandidateExclusionRecord`、`PrimaryCandidatePool` |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 6 检索后构建候选池并写入 ResearchBootstrapResult；不调用筛选 Agent |
| 新增 | `backend/tests/unit/services/test_primary_candidate_pool_service.py` | 分类、顺序、去重、required、饮食过滤、美食开关和空池测试 |
| 修改 | `backend/tests/integration/test_trip_coordinator_bootstrap.py` | 增加 Stage 6→7A 契约、required 保留、ignored 零影响和模式回归 |

本 Stage 不修改 AmapService、Provider、Stage 6 检索策略、DestinationResearchAgent、API 路由、legacy Agent、前端类型或页面。

## 4. 输入、输出与错误契约

### 4.1 输入边界

`PrimaryCandidatePoolService.build()` 只接收：

- Stage 6 已校验的 `PreferenceSearchResult`，包括请求顺序、hits、required POI 引用和查询记录；
- Stage 5 已规范化的 accepted constraints，用于核对 required constraint IDs 和执行 `dietary_exclusion`；
- 当前 `request_id`，必须与两个输入对象一致。

服务不得重新调用 AmapService、Provider 或 LLM，不得读取自由文本，也不得自行扩展检索关键词。`ignored_extra_requirements` 不作为构建输入；Coordinator 只原样保留其诊断，不允许其影响候选。

### 4.2 候选分类与入池规则

候选类型封闭为：

| `PrimaryCandidateKind` | 入池来源 | 规则 |
|------------------------|----------|------|
| `attraction` | Stage 6 target type 为 attraction、shopping 或 leisure 的真实 hit | 作为非餐饮主要点候选；保留原始 `SourcedPOI.poi_type`，不得改写事实类型 |
| `gourmet` | Stage 6 中 `food` 偏好查询命中的 restaurant，或 accepted required restaurant | 普通命中仅在显式包含 `food` 时入池；required 不受普通偏好开关限制；两者都须通过全部 dietary rules |

- shopping/leisure 是检索目标类别，不新增主要点位 kind；它们在候选层统一归为 `attraction`，命中偏好和原始 POI 类型仍完整保留，供 Stage 7B 判断。
- 非 `food` 查询意外返回 restaurant 时不进入主要点位池，记录 `unsupported_primary_type` 排除诊断；不得据 POI 类型猜测用户开启了美食偏好。
- `food` 查询返回非 restaurant 时不进入池并记录同一诊断，禁止把景点伪装为美食级餐点。
- required POI 使用其真实类型分类：非餐饮实体归为 `attraction`；餐饮实体归为 `gourmet`，即使未显式选择 `food` 也必须入池，因为 accepted 约束高于普通偏好。required 餐饮仍须通过 dietary rules；与 accepted 饮食排除冲突时返回业务不可行结果 `required_poi_excluded_by_dietary_rule`，不得静默删除任一约束或自行判断菜品例外。
- 本 Stage 不对营业时间做固定餐窗预筛：美食级候选尚未获得 `meal_role`。美食级午晚餐的餐窗预筛在 Stage 9A、最终实际区间校验在 Stage 11/12B；缺失营业时间保持原始 `unavailable` 状态。

### 4.3 稳定合并、required 与约束引用

- 普通候选按 Stage 6 `hits` 首次出现顺序入池；required POI 无论是否命中偏好都必须纳入，并按 accepted constraints 顺序追加尚未出现的实体。
- 实体键固定为 `(provider, source_id)`。同一实体只生成一个 `PrimaryCandidate`；普通 hit 与 required 重合时保留原位置，并将 `required=true`、合并全部关联 `constraint_ids`。
- `matched_preferences` 沿用 Stage 6 的用户顺序，不追加推测标签。required-only 候选允许为空。
- `constraint_ids` 只允许引用 accepted constraints，按 accepted constraints 顺序稳定去重；ignored ID、未知 ID 或同 ID 内容不一致时拒绝构建。
- 同实体出现不同事实快照时沿用 Stage 6 规则：核心身份字段冲突抛 `CandidatePoolError(error_code="candidate_source_identity_conflict")`；可选字段差异完整保留 Stage 6 已选定的事实引用，不拼接快照。
- required 身份是不可降级属性。后续 Stage 7B 必须保留，Stage 8 不得分配删除排名；本 Stage 只表达该属性，不预设日期或顺序。

### 4.4 饮食排除与审计记录

- 仅对准备作为 `gourmet` 入池的真实餐饮 POI执行 Stage 5 `DietaryRuleSet`；全部 accepted `dietary_exclusion` 按 accepted constraints 顺序执行，任一命中即排除。
- 排除判断只能使用规则表允许的来源字段，例如具体名称、原始 POI 类型/类别；不得调用 LLM、查询菜单或根据缺失字段反向推断安全。
- 普通 gourmet 命中规则时不入池，并生成 `CandidateExclusionRecord`：entity key、candidate kind、reason=`dietary_exclusion`、命中的 rule IDs 和 constraint IDs。记录不得复制完整 POI 或自由文本。
- 同一实体命中多个规则只产生一条排除记录，rule IDs 与 constraint IDs 稳定去重；测试和日志可以复核每个排除决定。
- `dietary_exclusion` 不过滤非餐饮主要点，也不涉及 Stage 9A 的便餐或 `lodging_side` 早餐；后两者必须在各自 Stage 再执行同一规则表。
- ignored 子句、未知规则或规则执行异常不得静默变成“未命中”；模型校验拒绝未知 rule ID，执行异常作为技术错误传播。

### 4.5 输出模型与派生字段

`PrimaryCandidatePool` 至少包含：

- `request_id`、city、`food_preference_enabled`；该字段只表示普通美食候选策略是否开启，false 时仍允许 accepted required restaurant 作为 gourmet 入池；
- 按稳定顺序的 `candidates`，每项只引用 `SourcedPOI` 的实体键/事实引用，包含 kind、`matched_preferences`、required、constraint IDs 和输入首次序号；
- `exclusions`，只记录确定性排除原因及规则/约束引用；
- `attraction_count`、`gourmet_count`、`required_count`、`total_count`，全部由模型根据 candidates 派生或交叉校验，调用方不得传入不一致值；
- `status=ready|business_infeasible` 和可选稳定 `reason_code`。

`ready` 表示候选池契约成立，不表示数量足够形成完整行程，也不表示 Stage 7B/8 已成功。合法空池或过滤后无普通候选仍是可信业务输入：

- 无 required 冲突时返回 `ready` 空池，由 Stage 7B/后续规划判定候选不足；
- required 餐饮与饮食硬约束冲突时返回 `business_infeasible`，不得进入 Stage 7B；未开启普通美食偏好本身不与 required 餐厅冲突；
- required 非餐饮 POI存在时，即使全部普通命中为空，也必须返回包含该 required 的 ready 池。

输出不是筛选结果或草案，不包含 Agent 理由、评分、日期、`day_part`、`meal_role`、`removal_rank`、酒店、便餐、路线、时间或预算。

### 4.6 技术错误、业务不可行与日志

- request_id/city 不一致、未知 constraint ID、事实身份冲突、未知 dietary rule 或模型不变量破坏属于技术/契约错误，立即停止并传播，不返回部分候选池。
- 合法空检索、普通候选全部被饮食规则过滤属于可解释业务输入，不冒充 Provider/解析错误，不设置 degraded。
- required 与 accepted 硬约束冲突返回 `business_infeasible`；它不是 API 技术异常，也不得通过虚构候选恢复。
- Stage 17 使用 stale 缓存时，候选保留输入事实的 freshness；本 Stage 不自行改变 `degraded`。
- 日志按 request_id 记录 `input_hit_count/candidate_count/attraction_count/gourmet_count/required_count/exclusion_count/status/reason_code/duration`，不得记录 Key、自由文本、完整 POI、完整 constraint 或完整上游 payload。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或功能开关；候选类型、顺序、required 合并和饮食规则执行顺序是受测试锁定的领域策略。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求不运行 Coordinator；显式 shadow 验证可执行 Stage 7A；new 仍为 reserved。
- CandidatePoolService 及规则表由装配层注入，不读取 Settings，不访问网络或 LLM。

## 6. 实施步骤

1. 在 research 模型中定义封闭 candidate kind、候选、排除记录和候选池不变量，先补序列化正反例。
2. 实现 attraction/shopping/leisure 与 gourmet 的确定性分类，拒绝输入目标类型和真实 POI 类型不一致的实体。
3. 实现 Stage 6 hit 的稳定入池和 `(provider, source_id)` 去重，完整保留匹配偏好与事实引用。
4. 按 accepted constraints 顺序合并 required POI，验证 required 身份、constraint IDs 和事实冲突规则。
5. 仅在 `food` 偏好开启时建立 gourmet 候选，对其执行全部 accepted dietary rules并生成脱敏排除记录。
6. 实现空池、required-only、普通候选全过滤和 required 硬约束冲突的不同状态语义。
7. 将 CandidatePoolService 接入 TripCoordinator 的 Stage 6 输出后，扩展 ResearchBootstrapResult；保持天气、ignored 和模式分发行为不变。
8. 执行 Stage 1～7A 定向测试、后端全量回归和前端构建，归档候选矩阵、饮食过滤与回滚证据。

## 7. 明确不做

- 不重新检索或扩展关键词，不并发、不分页、不缓存，不改变 Stage 6 的每偏好和总候选上限。
- 不调用 DestinationResearchAgent，不做评分、质量排序、数量裁剪或最终主要点位选择；这些属于 Stage 7B。
- 不生成日期、活动顺序、`day_part`、`meal_role`、建议时长或删除排名；这些属于 Stage 8。
- 不检索或筛选酒店、便餐，不建立空间锚点，不校验午晚餐固定餐窗；这些属于 Stage 9A 以后。
- 不依据天气替换候选；Stage 18 前天气对候选池零影响。
- 不修改 API、前端或 legacy 链路，不开放 new 模式，不删除 shadow 防护。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S7A-T01 | unit | attraction/shopping/leisure 命中 | 均进入 kind=attraction，原始类型和偏好标签不变 |
| S7A-T02 | unit | food + restaurant 命中 | food 开启时进入 kind=gourmet，事实字段保持来源引用 |
| S7A-T03 | unit | 未开启 food 但出现 restaurant | 不提升为主要点，记录 unsupported_primary_type |
| S7A-T04 | unit | food 查询返回非 restaurant | 不入池且有稳定排除诊断，不改写类型 |
| S7A-T05 | unit | 稳定顺序与重复 source ID | 首次位置不变，只保留一个候选并合并标签 |
| S7A-T06 | unit | required-only 非餐饮 POI | 追加进池、required=true、constraint IDs 完整 |
| S7A-T07 | unit | required 与普通 hit 重合 | 保留原位置和一个事实实体，升级 required 身份 |
| S7A-T08 | unit | 多个 required | 按 accepted constraints 顺序追加并稳定去重 |
| S7A-T09 | unit | required 引用 ignored/未知 constraint | 拒绝构建，不产生部分候选池 |
| S7A-T10 | unit | 同 source ID 核心身份冲突 | 抛 candidate_source_identity_conflict |
| S7A-T11 | unit | 单一 dietary rule 命中 gourmet | 候选被排除，记录 rule ID 和 constraint ID |
| S7A-T12 | unit | 同 gourmet 命中多个规则 | 只生成一条 exclusion，规则与约束顺序稳定 |
| S7A-T13 | unit | dietary rule + 非餐饮候选 | 非餐饮候选不受影响 |
| S7A-T14 | unit | required gourmet 被 dietary rule 排除 | status=business_infeasible，reason 稳定，不进入后续 |
| S7A-T15 | unit | required restaurant 但未开启 food | 仍以 gourmet、required=true 入池，普通餐厅命中仍不提升 |
| S7A-T16 | unit | 无 hit、无 required | 返回 ready 空池，计数均为 0，不标技术失败 |
| S7A-T17 | unit | 普通 gourmet 全部被过滤 | 返回 ready 空池及完整 exclusions，不伪造替代候选 |
| S7A-T18 | unit | required-only + 普通空结果 | ready 且 required 必然存在，计数派生正确 |
| S7A-T19 | unit | request_id/city/派生计数不一致 | 严格模型或服务拒绝 |
| S7A-T20 | integration | Stage 6→7A Coordinator | ResearchBootstrapResult 含正式候选池，查询记录/天气保持不变 |
| S7A-T21 | integration | ignored 额外要求 | 候选、排除、required 与无 ignored 基线一致 |
| S7A-T22 | regression | legacy/shadow/new 模式 | 普通 legacy/shadow 不构建候选池，new 仍 planner_mode_not_ready |
| S7A-T23 | regression | Stage 1～6 与前端 | Provider、模型、解析、约束、检索、shadow 测试和构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_primary_candidate_pool_service.py backend/tests/integration/test_trip_coordinator_bootstrap.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实 LLM。Stage 7A 必测用例数为 23，失败数必须为 0。

## 9. 人工验收

不适用：Stage 7A 不接入普通 API/UI，分类、required 合并、饮食过滤、顺序与状态语义均由固定 fixture 和 Fake Provider 输出自动化覆盖；显式 shadow 人工入口沿用 Stage 5，不重复建立页面验收。

## 10. 量化退出指标

- [ ] S7A-T01～S7A-T23 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 所有入池实体均具有真实 `(provider, source_id)` 和 Stage 3 合法事实引用；虚构或未知实体数为 0。
- [ ] 普通候选顺序与 Stage 6 首次命中顺序一致；重复实体数为 0，匹配偏好顺序稳定。
- [ ] accepted required POI 入池遗漏数为 0；required 与普通 hit 重合时事实实体仍只有 1 个，constraint IDs 丢失数为 0。
- [ ] food 关闭时普通 gourmet 候选数为 0，但 accepted required restaurant 仍以 gourmet 入池；food 开启时只有真实 restaurant 可进入 gourmet 池。
- [ ] 全部 accepted dietary rules 均执行；规则命中的普通 gourmet 入池数为 0，每个排除决定均可由 exclusion 与测试 ID 追溯。
- [ ] required 与饮食硬约束冲突时全部返回稳定 business_infeasible，不进入 Stage 7B，不触发虚构 fallback；普通美食开关不覆盖 required 优先级。
- [ ] 空检索、普通候选全过滤、required-only 和技术错误四类语义有独立测试，不互相冒充。
- [ ] 输出不包含 Agent 评分、日期、顺序、meal_role、酒店、便餐、路线、时间或预算；越界字段数为 0。
- [ ] Coordinator 不新增 AmapService/LLM 调用；查询保持 Stage 6 同步串行预算，并发任务数和默认测试公网调用数均为 0。
- [ ] legacy/shadow 普通请求行为不变，new 仍 reserved；Stage 1～6 回归与前端构建通过。
- [ ] 03 Stage 7A 每条退出条件及 02 的来源、required、饮食规则均映射到测试/证据 ID，新增未追踪 TODO 数为 0。

## 11. 迁移与回滚

- Stage 7A 只在显式新链路的内部 ResearchBootstrapResult 增加候选池；默认用户流量仍为 legacy，无持久化数据迁移。
- 回滚时移除 Coordinator 的 CandidatePoolService 调用及 Stage 7A 模型/测试，恢复 Stage 6 `PreferenceSearchResult` 作为 bootstrap 末端；Stage 1～6 行为和数据保留。
- 回滚不得删除 Stage 5 accepted/ignored constraints、required bindings、dietary rule 表或 Stage 6 查询记录。
- 回滚后 Stage 1～6 全量测试、legacy/shadow 模式测试和前端生产构建必须通过。
- Stage 7B 以后已依赖 PrimaryCandidatePool 时必须按依赖逆序回滚，不能只移除候选池构建。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_7a/test-summary.md` | 提交标识、三条命令、23 个用例结果、离线声明和失败数 |
| 候选构建矩阵 | `doc/opt_strategy/evidence/stage_7a/candidate-pool-matrix.md` | 分类、顺序、重复、required、空池和状态语义及测试 ID |
| 饮食过滤矩阵 | `doc/opt_strategy/evidence/stage_7a/dietary-filter-matrix.md` | food 开关、required 优先级、规则命中、多规则、required 冲突和排除记录 |
| 边界审计 | `doc/opt_strategy/evidence/stage_7a/stage-boundary-check.md` | Stage 6 输入、7A 输出及未越界进入 7B/8/9A 的检查结果 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_7a/rollback-check.md` | Coordinator 恢复点、依赖逆序要求、命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计状态：**工作包已细化，待 Stage 6～8 第二次跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 5、6 实际 Done；审计时重点复核 Stage 6 `PreferenceSearchResult` 到本 Stage `PrimaryCandidatePool` 的唯一归属、required 餐饮高于普通 food 偏好的优先级，以及 Stage 7B/8/9A 的职责边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
