# Stage 6：多偏好完整检索

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 6 |
| 目标 | 让 TripCoordinator 按用户选择顺序完整检索最多 3 个偏好，确定性合并真实 POI并保留命中标签，替代只使用第一个偏好的旧行为 |
| 对应优化项 | P0-3 多偏好完整检索 |
| 前置依赖 | Stage 5 |
| 后续阻断 | 未通过时不得开始 Stage 7A、7B、8、13A、18 |
| 默认流量影响 | 仅扩展显式新链路/bootstrap；legacy 与普通 shadow 请求仍不执行 Coordinator，默认 `/trip/plan` 响应不变 |

## 2. 当前代码基线与差异

- legacy `_build_attraction_query()` 只读取 `request.preferences[0]`，无偏好时使用“景点”；其余标签被静默丢弃。
- Stage 5 新链路只执行一次通用景点基线检索，用于证明结构化调用链可达，尚未遍历 `TravelPreference`。
- Stage 2 已将偏好限制为六类受控枚举、最多 3 个且不重复；Stage 3 `search_poi()` 支持 `page_size=1..20` 并返回真实 `SourcedPOI`。
- 当前没有偏好到检索关键词/POI 类别的固定映射、单偏好上限、合并总上限、命中偏好标签或跨查询去重规则。
- Stage 5 已单独绑定 accepted required POI；它们不能因偏好候选截断而丢失，也不能被伪装成普通偏好命中。
- 当前未定义“某个偏好合法空结果”和“某个偏好技术失败”的不同语义，容易出现静默缺失或错误 partial。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/services/preference_search_policy.py` | 冻结六类偏好的关键词、目标 POI 类型、查询顺序和数量上限 |
| 新增 | `backend/app/services/preference_search_service.py` | 同步串行调用 AmapService、按 source ID 合并、保留命中标签与查询诊断 |
| 新增 | `backend/app/services/preference_search_errors.py` | 定义 `PreferenceSearchError` 与 `source_identity_conflict`，不复用 Provider/解析错误类型 |
| 修改 | `backend/app/models/research.py` | 增加 `PreferenceHit`、`PreferenceSearchRecord`、`PreferenceSearchResult` 与 required POI 独立引用 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 用多偏好检索替换 Stage 5 通用景点基线；天气与约束流程保持不变 |
| 新增 | `backend/tests/unit/services/test_preference_search_policy.py` | 六类映射、无偏好策略和上限常量测试 |
| 新增 | `backend/tests/unit/services/test_preference_search_service.py` | 顺序、调用次数、去重、标签、截断、空结果和错误传播测试 |
| 修改 | `backend/tests/integration/test_trip_coordinator_bootstrap.py` | 增加 0～3 偏好、required 独立保留和 Stage 5 回归场景 |

本 Stage 不修改 AmapService、Provider、领域事实 parser、API 路由、legacy Agent、前端类型或页面。

## 4. 输入、输出与错误契约

### 4.1 固定偏好检索策略

| `TravelPreference` | 高德检索关键词 | 目标 POI 类型 | `matched_preferences` 标签 |
|--------------------|----------------|---------------|-----------------------------|
| `history_culture` | `历史文化景点` | attraction | history_culture |
| `natural_scenery` | `自然风光景点` | attraction | natural_scenery |
| `food` | `特色美食` | restaurant | food |
| `shopping` | `购物中心` | shopping | shopping |
| `art` | `美术馆` | attraction | art |
| `leisure` | `休闲娱乐` | leisure | leisure |

- 映射是确定性检索策略，不由 LLM 改写、扩词或补同义词；后续如需多关键词召回，必须在独立 Stage 修改策略和调用预算。
- 每个已选偏好只发起 1 次 `search_poi()`，固定 `city_limit=true`、`page_size=10`，按用户选择顺序同步串行执行。
- 用户偏好顺序是合并顺序的一部分，不按枚举、中文名称或返回数量重新排序。
- `food` 返回餐饮真实 POI，Stage 6 只标记 preference hit；是否进入美食级主要候选池、饮食过滤和数量硬规则由 Stage 7A 负责。

### 4.2 无偏好策略

- `preferences=[]` 时只发起 1 次通用检索：关键词 `热门景点`、目标类型 attraction、`city_limit=true`、`page_size=20`。
- 通用结果的 `matched_preferences=[]`，同时记录 `search_reason=default_general`；禁止制造不存在的“通用偏好”枚举。
- 无偏好不是跳过检索，也不得复用某个历史请求候选；缓存能力在 Stage 17 引入后仍须遵守同一逻辑键。

### 4.3 数量上限与稳定合并

- 有偏好时每个查询最多接收前 10 个合法 POI，偏好候选合并总上限固定为 30；最多 3 个偏好时无需额外丢弃任何单偏好已接收项。
- 无偏好通用候选上限固定为 20。
- 按 `(provider, source_id)` 稳定去重：首次出现决定候选位置，后续重复只按用户偏好顺序追加尚未存在的 `matched_preferences`。
- 同 source ID 的 name/address/location/poi_type 任一核心身份字段冲突时抛 `PreferenceSearchError(error_code="source_identity_conflict")`，不得静默保留某次结果；仅可选字段不同时完整保留首次出现的 SourcedPOI，并记录 `optional_field_difference` 脱敏诊断，不拼接两个快照的字段。
- required POI bindings 不计入 30 个偏好候选上限，保存在 `required_pois` 独立集合；Stage 7A 合并候选池时必须无条件纳入并标记 required。
- required POI同时出现在偏好结果时，偏好结果照常记录命中标签，但事实实体只保留一个 source ID；required 身份和 constraint IDs不得被覆盖。

### 4.4 输出模型

`PreferenceSearchResult` 至少包含：

- `request_id`、city、按请求顺序的 `requested_preferences`；
- 按首次出现顺序的 `hits`，每项引用真实 SourcedPOI、目标类型、`matched_preferences` 和首次查询序号；
- 独立 `required_pois` 引用，不复制/改写来源事实；
- 每次查询的 `PreferenceSearchRecord`：preference/search_reason、keyword、target_type、requested page_size、raw valid count、accepted count、empty、duration 和稳定诊断；
- `total_unique_hits`，必须等于 hits 长度，不由调用方传入任意值。

输出不是 Stage 7A 的正式 CandidatePool，不包含 Agent 评分、饮食过滤、required 合并结果、日期、顺序、meal_role 或天气决策。

### 4.5 空结果与错误

- 单个或全部偏好返回合法 `[]` 时仍返回成功 `PreferenceSearchResult`，对应 record 标记 `empty=true`；候选是否足以规划由 Stage 7A 以后判断。
- 任一偏好查询发生 ProviderError、AmapParseError 或 PreferenceSearchError 时立即停止后续偏好调用并传播技术错误；不得跳过失败偏好后伪装“已完整检索”。
- 合法空结果不设置 degraded；Stage 17 使用 stale 缓存成功时才按全局语义设置数据降级。
- ignored extra requirements 不进入偏好映射或查询；accepted must-visit 只使用 Stage 5 已绑定 required POI，不改变普通偏好调用次数。
- 日志按 request_id 记录 `preference_index/preference/keyword/target_type/duration/result_count/status`，不得记录 Key、完整 payload、完整 POI 列表或用户自由文本。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或功能开关；关键词、目标类型、page_size=10、总上限 30 和无偏好上限 20 是受测试锁定的代码策略。
- 沿用 `TRIP_PLANNER_MODE`：legacy/shadow 普通请求仍不运行 Coordinator，显式 shadow 验证可执行 Stage 6 新逻辑，new 仍为 reserved。
- AmapService、Provider 和时钟继续由装配层注入；PreferenceSearchService 不读取 Settings。

## 6. 实施步骤

1. 建立 PreferenceSearchPolicy 常量表和六类/无偏好正反例测试，冻结关键词、类型和数量上限。
2. 定义 PreferenceSearchResult 相关严格模型，复用 Stage 2/3 的 SourcedPOI 与 request_id，不复制事实字段为可修改文本。
3. 实现无偏好单查询路径，验证 page_size=20、空结果和 default_general 记录。
4. 实现 1～3 偏好同步串行调用，固定每偏好 page_size=10，记录调用顺序和查询诊断。
5. 实现 source ID 稳定去重、matched_preferences 有序合并、核心身份冲突和可选字段差异诊断规则。
6. 将 Stage 5 Coordinator 的通用景点基线替换为 PreferenceSearchService，并独立透传 required POI bindings。
7. 补齐技术错误立即停止、合法空结果继续、ignored 零影响以及 Stage 5 shadow/legacy 回归测试。
8. 执行 Stage 1～6 定向测试、后端全量回归和前端构建，归档策略、调用和回滚证据。

## 7. 明确不做

- 不做多关键词扩展、模糊召回、并发查询、分页追取、缓存或基于热度/评分的重排。
- 不筛选主要点位，不执行饮食规则，不把 restaurant 自动提升为美食级候选；属于 Stage 7A/7B。
- 不调用 DestinationResearchAgent 对偏好候选评分，不生成 PrimaryDraft、酒店、便餐、路线、时间轴或 TripPlan。
- 不处理天气驱动替换，Stage 18 前天气字段对偏好检索和合并零影响。
- 不修改 API/前端/legacy 链路，不开放 new 模式，不删除 Stage 5 shadow 防护。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S6-T01 | unit | 六类偏好映射 | keyword、target_type、page_size 精确匹配固定表 |
| S6-T02 | unit | 未知/重复偏好 | Stage 2 请求模型拒绝，服务不发起调用 |
| S6-T03 | unit | 无偏好 | 仅 1 次热门景点查询，page_size=20，matched_preferences 为空 |
| S6-T04 | unit | 1、2、3 个偏好 | 调用数分别为 1、2、3，顺序与用户输入一致且同步串行 |
| S6-T05 | unit | 每偏好返回超过 10 项 | 只接受来源顺序前 10 项，record 记录 raw/accepted count |
| S6-T06 | unit | 三偏好总量 | unique hits 不超过 30，不因合并改变首次出现顺序 |
| S6-T07 | unit | 同 source ID命中多个偏好 | 只保留一个事实实体，matched_preferences 按用户顺序合并 |
| S6-T08 | unit | 同 source ID 核心身份冲突 | 抛 PreferenceSearchError/source_identity_conflict，不继续后续偏好调用 |
| S6-T09 | unit | 同 source ID 可选字段差异 | 完整保留首次事实快照并记录诊断，不拼接两个快照字段 |
| S6-T10 | unit | 单个/全部偏好合法空结果 | 成功返回，empty record 完整，不标 degraded |
| S6-T11 | unit | 第二个偏好发生 ProviderError/AmapParseError | 立即停止，第三个偏好调用数为 0，技术错误原样传播 |
| S6-T12 | integration | accepted required POI + 重复偏好命中 | required 独立保留且事实去重，constraint IDs不被偏好标签覆盖 |
| S6-T13 | integration | ignored 额外要求 | 调用关键词/次数/结果与无 ignored 基线完全一致 |
| S6-T14 | integration | Stage 5 Coordinator 替换基线 | ResearchBootstrapResult 使用 PreferenceSearchResult，天气/约束流程不变 |
| S6-T15 | regression | legacy/shadow/new 模式 | legacy/shadow 普通请求 Coordinator 调用数 0，new 仍返回 planner_mode_not_ready |
| S6-T16 | regression | Stage 1～5 与前端 | Provider、模型、解析、约束、shadow 测试和生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_preference_search_policy.py backend/tests/unit/services/test_preference_search_service.py backend/tests/integration/test_trip_coordinator_bootstrap.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实 LLM。Stage 6 必测用例数为 16，失败数必须为 0。

## 9. 人工验收

不适用：Stage 6 不接入普通 API/UI，偏好映射、调用顺序、数量上限、去重和错误行为均由 FakeAmapProvider 自动化覆盖；显式 shadow 的人工入口沿用 Stage 5，不重复建立页面验收。

## 10. 量化退出指标

- [ ] S6-T01～S6-T16 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 六类偏好映射和无偏好策略精确固定；1～3 个偏好的 AmapService 调用数分别为 1～3，无偏好为 1。
- [ ] 每偏好 accepted 上限为 10、偏好 unique hits 总上限为 30、无偏好上限为 20，任何路径均不超限。
- [ ] 所有已选偏好都有且只有一条 PreferenceSearchRecord；合法空结果也保留记录，不被视为技术错误。
- [ ] 重复 source ID 只保留一个事实实体，matched_preferences 顺序稳定；核心身份冲突不被静默覆盖，可选字段差异不产生混合快照。
- [ ] required POI 不受偏好上限影响，constraint IDs 和 required 身份不被覆盖；ignored 对调用和结果零影响。
- [ ] 任一技术错误立即停止后续检索并原样传播，不返回“完整检索”假结果，不触发虚构 fallback。
- [ ] 查询保持同步串行；并发任务数为 0，默认测试公网调用数为 0。
- [ ] legacy/shadow 普通请求行为不变，new 仍 reserved；Stage 1～5 回归与前端构建通过。
- [ ] 03 Stage 6 每条退出条件和 02 多偏好完整检索规则均映射到测试/证据 ID，新增未追踪 TODO 数为 0。

## 11. 迁移与回滚

- Stage 6 只替换显式新链路中的 Stage 5 通用景点基线；默认用户流量仍为 legacy，无数据迁移。
- 回滚时恢复 Coordinator 对 Stage 5 单次通用景点检索的调用，删除 PreferenceSearchService/Policy 与新增模型/测试；Stage 1～5 文件和行为保留。
- 回滚后 Stage 1～5 全量测试、legacy/shadow 模式测试和前端生产构建必须通过。
- Stage 7A 以后已依赖 PreferenceSearchResult 时必须按依赖逆序回滚，不能单独恢复通用基线。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_6/test-summary.md` | 提交标识、三条命令、16 个用例结果、离线声明和失败数 |
| 策略矩阵 | `doc/opt_strategy/evidence/stage_6/preference-policy.md` | 六类/无偏好映射、page_size、总上限和测试 ID |
| 调用与合并矩阵 | `doc/opt_strategy/evidence/stage_6/search-merge-matrix.md` | 0～3 偏好调用顺序、空结果、重复、冲突、错误停止和测试 ID |
| 回滚验证 | `doc/opt_strategy/evidence/stage_6/rollback-check.md` | Coordinator 恢复点、依赖逆序要求、命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计状态：**工作包已细化，R2～R10 设计复核通过；执行 Ready 待 R1**。R1 必须等待 Stage 5 实际 Done；第二次跨 Stage 审计已确认 Stage 5 ResearchBootstrapResult、Stage 7A CandidatePool 和 required POI 的归属边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
