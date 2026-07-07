# Stage 5：额外要求解析与标准化数据调用链 shadow 接入

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 5 |
| 目标 | 建立最小 TripCoordinator 与结构化额外要求解析链路，使新链路直接消费 AmapService 真实对象，并以可回滚 shadow 方式脱离 `[TOOL_CALL:...]` 文本工具调用 |
| 对应优化项 | P0-2 标准化数据调用链、P0-4 结构化输出，以及 P0-3 的调用链前置 |
| 前置依赖 | Stage 1、2、3、4 |
| 后续阻断 | 未通过时不得开始 Stage 6～16、18、23、28、31B/31C |
| 默认流量影响 | 默认 `legacy`；普通 `/trip/plan` 继续返回旧链路结果，新链路只由离线测试或显式本地 shadow 验证入口执行 |

## 2. 当前代码基线与差异

- 当前 `/api/trip/plan` 直接获取全局 `MultiAgentTripPlanner` 并在 async 路由中同步执行，所有异常统一转为 500；没有 Coordinator、dispatcher、request_id 或链路模式边界。
- `MultiAgentTripPlanner` 内部直接创建 MCPTool，景点/天气/酒店 Agent 依赖 `[TOOL_CALL:...]` 文本格式；AmapService 的结构化对象没有进入规划链路。
- `_build_attraction_query()` 只使用 `request.preferences[0]`，其余偏好丢失；Stage 5 只移除新链路的文本工具调用，完整多偏好检索留给 Stage 6。
- `free_text_input` 当前整段拼入 Planner Prompt，没有子句拆分、accepted/ignored、封闭约束或 required 语义；模糊要求可能直接影响 LLM 生成事实。
- Planner 响应通过代码块/首尾花括号猜取 JSON；失败后 `_create_fallback_plan()` 生成虚构景点、默认北京附近坐标和泛化餐饮。新链路禁止继承该 fallback。
- `llm_service.py` 只暴露 HelloAgentsLLM 单例，没有 JSON mode/response_format 适配、schema 校验、温度和有限重试边界。
- 当前不存在 POI 唯一匹配、饮食排除规则表、最小 ResearchBootstrapResult、shadow 差异摘要或 `TRIP_PLANNER_MODE` 配置。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/coordinators/trip_coordinator.py` | 最小 bootstrap 编排：解析额外要求、绑定 required POI、调用结构化 AmapService、产出内部研究结果；不生成 TripPlan |
| 新增 | `backend/app/coordinators/__init__.py` | 只导出 Coordinator 稳定创建入口 |
| 新增 | `backend/app/agents/destination_research_agent.py` | 使用结构化 LLM 适配器将编号子句解析为四类 accepted/ignored 结果，不直接调用高德工具 |
| 新增 | `backend/app/services/structured_llm.py` | 封装 JSON mode/response_format、温度、Pydantic 校验和有限重试；不实现业务 Prompt |
| 新增 | `backend/app/services/extra_constraint_service.py` | 确定性子句切分、稳定 ID、约束归一化、显式时间关联 required 目标和 parse_failed 组装 |
| 新增 | `backend/app/services/poi_matcher.py` | 城市/adcode 限定、名称规范化完全匹配、来源 ID 去重和唯一性判定 |
| 新增 | `backend/app/services/dietary_rules.py` | 封闭饮食排除规则及对真实餐饮 POI 的确定性匹配；不做菜单级推理 |
| 新增 | `backend/app/services/trip_planner_dispatcher.py` | 按 `TRIP_PLANNER_MODE` 选择 legacy 行为；Stage 5 不开放 new 为默认入口 |
| 新增 | `backend/app/models/research.py` | 定义内部 `ResearchBootstrapResult`、bootstrap ready/failed、shadow 摘要和结构化数据引用 |
| 修改 | `backend/app/config.py` | 增加并启动校验 `trip_planner_mode`，默认 legacy |
| 修改 | `backend/app/api/routes/trip.py` | 改由 dispatcher 调用 legacy；生成/传播 request_id；普通 shadow 请求不得自动双跑新链路 |
| 新增 | `backend/tests/unit/services/test_extra_constraint_service.py` | 子句、ID、归一化、parse_failed 和关联约束测试 |
| 新增 | `backend/tests/unit/services/test_poi_matcher.py` | 城市、名称、分店、去重与唯一性测试 |
| 新增 | `backend/tests/unit/services/test_dietary_rules.py` | 固定规则、误命中边界和住宿侧早餐例外测试 |
| 新增 | `backend/tests/contract/test_destination_research_agent.py` | FakeLLM 下的 JSON mode、schema、重试和四类约束测试 |
| 新增 | `backend/tests/integration/test_trip_coordinator_bootstrap.py` | FakeAmapProvider + FakeLLM 的最小新链路测试，不访问公网 |
| 新增 | `backend/tests/api/test_trip_planner_mode.py` | legacy/shadow/reserved-new、旧响应不受 shadow 失败影响和 request_id 测试 |
| 新增 | `backend/tests/live/verify_stage_5_shadow.py` | 显式本地 shadow 验证入口；不被 pytest 默认收集，不返回给普通用户 |

旧 `trip_planner_agent.py` 本 Stage 保留且不扩展能力，只作为 legacy 回滚入口。任何新模块不得导入其中的 Prompt、MCPTool 或 fallback。

## 4. 输入、输出与错误契约

### 4.1 最小 Coordinator 流程

```text
TripRequest + request_id
  → deterministic clause splitter
  → DestinationResearchAgent（结构化约束）
  → accepted 约束归一化
  → required POI 精确绑定 / dietary rule 准备
  → AmapService（通用景点基线 + 请求日期天气）
  → ResearchBootstrapResult（内部对象，不是 TripPlan）
```

- 空 `free_text_input` 不调用 LLM，直接得到 accepted/ignored 均为空的 parse result。
- Stage 5 只执行一次通用景点基线检索和请求日期天气查询，用于证明结构化数据链可达；不循环处理全部偏好，不检索酒店/便餐，不形成候选池或日程。
- 酒店和餐饮只通过 FakeAmapProvider 契约验证同一结构化 `search_poi()` 可按类型调用；真实锚点检索分别留给 Stage 7A/9A。
- `ResearchBootstrapResult` 只包含 request_id、bootstrap status、accepted constraints、ignored requirements、required POI bindings、结构化 POI/天气引用及错误/诊断摘要；不得包含伪装成正式结果的 TripPlan。
- `bootstrap=failed` 只用于已确认的业务失败，例如 accepted 必打卡无法唯一绑定；Provider/核心结构化解析技术失败继续抛稳定错误，不转换为 failed。

### 4.2 子句切分与 Agent 输入

- 先用确定性规则按换行、中文/英文句号、分号和逗号切分非空子句，保留原顺序并分配请求内 `clause_001...`；不按“和/以及/并且”等词语继续猜分，仍含多个动作的子句由 Agent 判为 ambiguous，不能只执行其中一部分。
- 子句原文只在当前内存 Prompt 中使用，不写入日志、fixture、TripPlan、shadow 摘要或证据；自动化测试只使用虚构、脱敏文本。
- 单次 Agent 调用接收全部编号子句并必须逐项返回相同 clause ID；缺项、重复项、未知项或额外项均属于非法结构化输出。
- Agent 只分类和提取执行字段，不搜索 POI、不生成 source ID、不判断餐厅菜单、不修改请求日期。
- 使用 temperature=0 和 JSON mode/response_format；返回字符串必须直接 `json.loads` 后由 Stage 2 `extra="forbid"` 模型校验，禁止正则或代码块猜取 JSON。

### 4.3 有限重试与 parse_failed

- 首次调用失败后最多重试 2 次，总调用上限为 3；仅 schema/JSON 非法、clause 对账失败和结构化输出暂时错误可重试。
- 每次重试向同一输入附加稳定、脱敏的校验错误摘要，不回显完整 Prompt 或用户原文，不改变 temperature。
- 任一合法响应即停止重试。三次均失败时，不让整次规划技术失败：每个输入 clause 生成 ignored/parse_failed，随后按无额外约束继续 bootstrap。
- LLM 配置缺失、认证、限流或超时等技术异常也遵循本解析例外的有限尝试；耗尽后按 clause 记 parse_failed，并记录脱敏 `parse_diagnostic`，但不得使用仅属于最终计划数据降级的 `degraded` 字段。该例外只适用于额外要求解析，不得复用于后续 PrimaryDraft/ItineraryDraft 等核心输出。
- 空输入调用次数必须为 0；一次成功为 1；两次失败后成功为 3；耗尽固定为 3，禁止无界重试。

### 4.4 四类约束归一化

| 类型 | accepted 必需字段 | ignored 条件 | 执行语义 |
|------|-------------------|--------------|----------|
| `must_visit_poi` | query、city、scope=trip | 对象不明确、缺 city、语义模糊 | 触发真实 POI 精确匹配并绑定 required |
| `dietary_exclusion` | 受控 rule_id | 无受控规则、泛化健康建议 | 由固定规则过滤系统选择的真实餐饮 POI |
| `explicit_time_constraint` | query、city、trip 内 date、带 +08:00 start/end | 字段不全、窗口非法、日期越界 | 同时生成关联 must_visit required 目标；二者必须共同满足 |
| `mobility_constraint` | 可验证的 max_walking_segment_meters | 完整无障碍等数据源不可验证语义 | 与系统 2000m 上限共同形成更严格有效上限 |

- constraint ID 使用规范化类型与执行字段的 canonical JSON 做 SHA-256，取前 20 个十六进制字符并加 `con_` 前缀；不得包含原始自由文本、随机值或当前时间。
- explicit time 自动生成的 must_visit constraint 使用独立稳定 ID，并通过 `related_constraint_ids` 双向关联；任一绑定失败时 bootstrap failed。
- mobility 用户上限与系统 2000m 上限取较小值作为 effective limit，原始规范化用户数值仍保留；它不在 Stage 5 查询或选择路线。
- ignored 只保留 clause ID、稳定 reason 和可选受控类型提示，不保留原文；ignored 不得进入 Amap 查询、accepted constraints 或下游 Prompt。

### 4.5 饮食排除规则

- 首版规则表冻结为：`avoid_spicy` 与 `avoid_seafood`。其他饮食语义在没有独立受控规则前进入 ignored/unsupported，不由 LLM 自由创建 rule_id。
- `avoid_spicy` 命中来源类型/名称中的川菜、湘菜、重庆火锅、麻辣烫等明确高风险类别；不因单店可能提供不辣菜恢复候选。
- `avoid_seafood` 命中海鲜、水产、鱼生、刺身等明确主营类别/名称。它只是确定性餐厅类型过滤，不能宣称完成菜单级过敏安全验证；最终提示必须要求用户向商家复核。
- 规则只检查来源 POI 的规范化 name/raw_type，不检查 LLM 描述；命中任一规则即排除，并记录 rule ID，不记录用户原文。
- 规则覆盖系统选择的美食级和便餐级真实 POI，不覆盖 `lodging_side` 早餐；Stage 5 只交付规则函数和 fake POI 验证，实际候选过滤在 Stage 7A/9A 接入。

### 4.6 必打卡与时间目标唯一匹配

- AmapService 按 request_id + query + city 执行真实检索；先按来源 city/adcode 限定，再按 source ID 去重，最后做名称完全匹配。条目缺 city/adcode 时，仅当 Stage 3 query_scope 可证明 Provider 严格执行 `city_limit=true` 才视为城市内，否则不得参与唯一匹配。
- 名称规范化仅执行 Unicode NFKC、首尾/连续空白归一和 ASCII casefold；不删除分店后缀、括号、标点或行政区文字，避免把不同实体错误合并。
- 规范化名称完全相等且唯一 source ID 时绑定成功；0 个匹配为 `required_poi_not_found`，多个不同 ID 为 `required_poi_ambiguous`。两者均使 bootstrap failed，不做模糊匹配、不让 LLM 选第一条。
- 同名不同分店视为不同实体；多个结果经 source ID 去重后仍大于 1 即 ambiguous。
- 绑定结果只保存真实 source ID、constraint IDs 和 required=true；POI 名称、地址、坐标继续从 Stage 3 来源事实回填，不复制后允许分叉。

### 4.7 shadow 与 dispatcher

- `legacy`：普通 `/trip/plan` 只执行旧链路；不运行 Coordinator。
- `shadow`：普通 `/trip/plan` 仍只执行旧链路，不因模式值自动双跑外部调用。新链路只由 `verify_stage_5_shadow.py` 或进程内显式验证函数调用。
- `new`：Stage 5 视为 reserved；dispatcher 收到普通规划请求时使用 Stage 2 `ApiErrorResponse` 返回 503 + `planner_mode_not_ready`，不得提前让不完整 bootstrap 充当正式 TripPlan。Stage 16 验收后才允许 new。
- shadow 输出不进入普通 API 响应，只向显式验证调用方返回脱敏摘要：request_id、约束数量/类型、ignored reason 计数、required 绑定状态、结构化数据数量、阶段耗时和稳定错误类别；是否归档由人工验收步骤决定，不在运行期自动写文件。
- Stage 5 尚无新 TripPlan，禁止 JSON 全文对比旧计划。shadow 只验证标准数据链和约束语义；规划质量对比从 Stage 13A 后开始。
- shadow 新链路失败不得改变旧链路响应、HTTP 状态或前端 sessionStorage；不得后台持续执行或保存结构化行程。

### 4.8 错误与日志

- 请求 schema 错误仍由现有 FastAPI 422 处理；Stage 5 不提前完成 Stage 15 的全局 API 错误映射。
- accepted required POI 无法唯一匹配是可信业务失败，形成 bootstrap failed；ProviderError 或 AmapParseError/invalid_response 是技术错误，二者不得互换。
- `BootstrapStatus=ready|failed` 只属于 Stage 5 内部研究结果，不等同于 `PlanStatus`；后续正式 Coordinator 必须将 required 绑定失败映射为整单 `PlanStatus.failed`，不得映射为 partial。
- 额外要求解析耗尽是 parse_failed ignored 例外；不得导致 bootstrap failed，也不得触发虚构 fallback。
- 日志统一传播 request_id 和 stage=`constraint_parse|required_poi_bind|baseline_poi|weather|shadow_summary`，记录耗时/状态但不记录 Key、完整 Prompt、自由文本、完整上游 payload 或 POI 列表。

## 5. 配置与功能开关

| Settings 字段 | 环境变量 | 允许值与 Stage 5 行为 |
|---------------|----------|------------------------|
| `trip_planner_mode` | `TRIP_PLANNER_MODE` | `legacy/shadow/new`；默认 legacy；非法值启动失败；new 在 Stage 16 前为 planner_mode_not_ready |

- 不新增 LLM secret 同义配置；沿用现有 LLM Settings/HelloAgents 环境读取，Stage 33 再统一清理遗留重复读取。
- 结构化解析重试上限固定为本工作包的 2 次重试，不开放环境变量，避免测试和运行语义漂移。
- shadow 验证入口仅本地显式运行，不提供公开 HTTP 开关，不写入前端配置。
- 回滚目标值为 legacy；模式切换不迁移数据，也不设置 degraded=true。

## 6. 实施步骤

1. 增加 `TRIP_PLANNER_MODE` 启动校验与 dispatcher，先证明 legacy 响应、状态码和前端契约不变，shadow 普通请求不双跑。
2. 建立 StructuredLLMAdapter 与 FakeLLM 契约，固定 JSON mode、temperature=0、`extra="forbid"`、clause 对账和总调用上限 3。
3. 实现确定性子句切分、constraint ID、四类归一化、ignored/parse_failed 和 explicit time 关联 required 约束。
4. 实现饮食规则表与 POI 精确匹配器，覆盖城市/adcode、名称规范化、分店、来源 ID 去重和 0/1/N 结果。
5. 建立 DestinationResearchAgent；Agent 只解析编号子句，不挂 MCPTool，不接触候选筛选或计划生成。
6. 建立最小 TripCoordinator bootstrap，串行调用约束解析、required 绑定、通用景点基线和天气；输出内部 ResearchBootstrapResult。
7. 增加显式本地 shadow 验证入口与脱敏摘要，验证新链路失败不影响 legacy；保留 reserved-new 防护。
8. 执行 Stage 1～5 定向测试、后端全量回归和前端构建，归档约束、匹配、错误、shadow 和回滚证据。

## 7. 明确不做

- 不实现完整多偏好检索、候选上限/合并/命中标签；属于 Stage 6。
- 不建立正式景点/美食候选池和 Agent 筛选；属于 Stage 7A/7B。
- 不生成 PrimaryItineraryDraft、完整 ItineraryDraft、TripPlan、路线选择、时间轴、预算或修复循环。
- 不检索/选择连续酒店和便餐，不执行饮食规则对真实完整候选集的最终过滤。
- 不让普通 shadow 请求双跑，不开放 new 默认链路，不删除旧 Agent 或虚构 fallback；fallback 必须在 Stage 16 前清除，但本 Stage 只保证新链路不调用它。
- 不实现缓存、SSE、数据库、对话修订、前端页面或生产部署配置。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S5-T01 | unit | 空、单句、多句及换行/中英文标点 | 句号/分号/逗号切分后顺序和 clause ID 稳定；空输入不调用 LLM |
| S5-T02 | contract | 四类 accepted 与 ignored 输出 | JSON mode、temperature=0、封闭联合和 `extra="forbid"` 生效 |
| S5-T03 | contract | clause 缺失、重复、未知、额外字段和非法 JSON | 输出拒绝并进入有限重试 |
| S5-T04 | contract | 首次成功、第三次成功与三次耗尽 | 调用数分别为 1、3、3；耗尽后每 clause 为 ignored/parse_failed |
| S5-T05 | unit | constraint ID 与 explicit time 关联 | 固定输入 ID 稳定；字段变化 ID 变化；关联 must_visit 独立且双向引用 |
| S5-T06 | unit | 时间字段不全、越界日期和 naive 时间 | ignored/incomplete 或非法输出，不猜日期/时区 |
| S5-T07 | unit | mobility 可验证上限与无障碍文本 | 前者 accepted 并取更严格有效值；后者 ignored/unsupported |
| S5-T08 | unit | avoid_spicy 正反例 | 固定类别/名称命中；非高风险实体不误排；不做菜单反推 |
| S5-T09 | unit | avoid_seafood 与过敏提示 | 主营海鲜类别命中；结果明确要求商家复核，不宣称菜单安全 |
| S5-T10 | unit | lodging_side 早餐 | 两类饮食规则均不校验或排除住宿侧早餐 |
| S5-T11 | unit | POI 城市/adcode、名称规范化与来源 ID 去重 | 仅城市内完全匹配进入唯一性计算，重复 ID 稳定去重 |
| S5-T12 | unit | 必打卡 0/1/N 匹配及同名分店 | 1 成功绑定；0 为 not_found；N/分店为 ambiguous；不模糊选第一条 |
| S5-T13 | integration | 无额外要求 bootstrap | 不调用 LLM；结构化通用 POI/天气进入 ready 结果 |
| S5-T14 | integration | 明确约束 + FakeAmapProvider | accepted/ignored、required binding 和来源对象引用正确 |
| S5-T15 | integration | parse_failed + Amap 正常 | bootstrap 继续 ready，ignored 对查询和结果零影响 |
| S5-T16 | integration | required 不唯一与 ProviderError | 前者 bootstrap failed，后者传播技术错误，均无 fallback TripPlan |
| S5-T17 | api | legacy 模式 | 只调用旧链路，响应 schema/状态不变，新 Coordinator 调用数为 0 |
| S5-T18 | api | shadow 模式普通请求 | 仍只调用旧链路；新链路失败不影响响应且调用数为 0 |
| S5-T19 | api | 非法模式与 reserved new | 非法值启动失败；new 返回 planner_mode_not_ready，不返回 bootstrap 假计划 |
| S5-T20 | integration | 显式 shadow 验证 | 只生成脱敏摘要，不写 API 响应/TripPlan，不记录自由文本或完整 POI |
| S5-T21 | contract | 新链路依赖边界 | DestinationResearchAgent 无 MCPTool/TOOL_CALL；Coordinator 只调用 AmapService，不导入 legacy planner |
| S5-T22 | regression | Stage 1～4 与前端 | Provider、模型、解析测试及生产构建不回退 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/unit/services/test_extra_constraint_service.py backend/tests/unit/services/test_poi_matcher.py backend/tests/unit/services/test_dietary_rules.py backend/tests/contract/test_destination_research_agent.py backend/tests/integration/test_trip_coordinator_bootstrap.py backend/tests/api/test_trip_planner_mode.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，默认测试不得访问公网或真实 LLM。Stage 5 必测用例数为 22，失败数必须为 0；live shadow 只在本地显式执行。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---------|------|------|
| S5-M01 | 本地 legacy 模式生成一份当前 Demo 行程 | 页面与旧响应可继续使用，新 Coordinator 未执行 |
| S5-M02 | 本地 shadow 模式通过普通页面生成行程 | 页面仍使用 legacy；浏览器请求不因 shadow 产生第二次外部规划调用 |
| S5-M03 | 显式运行 `verify_stage_5_shadow.py` 的脱敏样本 | 只输出约束/数据计数和错误类别摘要，无自由文本、Key、完整 POI 或 TripPlan |

实际结果进入 `manual-check.md`。Stage 5 不比较新旧行程质量，因为新链路尚未生成 TripPlan。

## 10. 量化退出指标

- [ ] S5-T01～S5-T22 全部通过，失败数为 0；S5-M01～S5-M03 全部通过；三条标准命令退出码为 `0`。
- [ ] 四类约束均有 accepted/ignored 正反例；空输入 LLM 调用数 0，单次成功 1，最大总调用数 3。
- [ ] parse_failed 耗尽后每个 clause 都进入 ignored，bootstrap 继续且下游查询不受 ignored 影响。
- [ ] explicit time 同时生成关联 required 目标；字段不全 ignored，目标无法唯一绑定时 bootstrap failed。
- [ ] 必打卡匹配严格按城市/adcode、规范化完全名称和 source ID 去重；0/1/N 与分店语义稳定，无模糊首选。
- [ ] avoid_spicy/avoid_seafood 规则确定性通过，住宿侧早餐不受过滤，海鲜规则不宣称菜单级过敏安全。
- [ ] 新链路中 `[TOOL_CALL:...]`、Agent MCPTool、正则 JSON 猜取和虚构 fallback 调用数均为 0。
- [ ] legacy/shadow 普通请求的新 Coordinator 调用数均为 0；reserved new 不返回不完整计划；显式 shadow 失败不改变旧响应。
- [ ] Coordinator 的 POI/天气事实全部来自 AmapService 结构化对象；技术错误不伪装业务 failed/空结果。
- [ ] 日志、错误、shadow 摘要和证据中的 Key、完整 Prompt、自由文本、完整 payload/POI 命中数为 0。
- [ ] Stage 1～4 回归和前端构建通过，新增未追踪 TODO 数为 0；03 Stage 5 每条退出条件均映射到测试/人工证据。

## 11. 迁移与回滚

- 默认模式为 legacy，Stage 5 无用户数据迁移。回滚首先将 `TRIP_PLANNER_MODE=legacy`，验证普通请求不加载 Coordinator。
- 删除 dispatcher 接入、Coordinator/新 Agent/规则服务和 Stage 5 测试前，保留 Stage 1～4 的 Provider、模型和 AmapService，不得回滚真实数据基础。
- 回滚后旧 `/trip/plan`、健康检查和前端生产构建必须通过；不得以启用旧虚构 fallback 作为新链路回滚验收依据，只确认 legacy 基线仍可运行。
- shadow 摘要不保存 TripPlan 或用户文本，无持久化清理；已归档的脱敏失败证据保留。
- Stage 6 以后已依赖 Coordinator 时必须按依赖逆序回滚，不能单独删除 Stage 5 骨架。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_5/test-summary.md` | 提交标识、三条命令、22 个用例结果、离线声明和失败数 |
| 约束矩阵 | `doc/opt_strategy/evidence/stage_5/constraint-matrix.md` | 子句 → accepted/ignored → 执行字段 → Amap 动作 → 测试 ID |
| 匹配/规则矩阵 | `doc/opt_strategy/evidence/stage_5/rule-matrix.md` | 名称唯一性、0/1/N、分店、饮食规则、早餐例外和测试 ID |
| shadow 摘要 | `doc/opt_strategy/evidence/stage_5/shadow-summary.md` | 模式、显式入口、计数/耗时/错误类别及脱敏检查，不含用户原文和 TripPlan |
| 人工验收 | `doc/opt_strategy/evidence/stage_5/manual-check.md` | S5-M01～M03 的步骤、实际结果和结论 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_5/rollback-check.md` | legacy 恢复、依赖边界、命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计复核结论：**R2～R10 通过首轮跨 Stage 审计；执行 Ready 待 R1**。R1 必须等待 Stage 1～4 实际 Done，并在 Stage 5 开工前再次确认结构化 AmapService、领域模型和错误契约未变化；当前不表示 Ready、Done、设计冻结或允许修改工程代码。
