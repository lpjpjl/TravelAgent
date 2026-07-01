# 旅行 Agent 优化实施计划

> 依据：`02_opt_plan.md`
>
> 文档状态：已完成与 02 文档的调用链、两次草案、连续住宿、封闭额外约束、修复职责、Stage 依赖和验收口径对齐；本文定义开发顺序，不改变 02 文档中的产品优先级。

---

## 一、实施目标与编排原则

本轮实施的核心目标是：先消除虚构数据和脆弱工具调用，再建立可执行、可追溯的结构化行程，随后补齐可靠性与用户体验，最后接入对话式迭代和低优先级工程治理。

### 1.1 排期规则

1. **小步快跑**：每个 Stage 只交付一个主要能力；一个 Stage 内不得同时重写 Agent、服务层和前端主流程。
2. **阶段可验收**：每个 Stage 都必须有与风险匹配的验证方式、明确输入输出和独立验收条件；确定性后端逻辑使用轻量测试，前端集成能力可采用记录化人工验收；未通过退出条件不得进入依赖它的 Stage。
3. **先契约后实现**：先确定 schema、状态和错误语义，再替换默认调用链，避免前后端反复返工。
4. **事实链优先**：外部事实必须来自 AmapService 或可追溯缓存；Agent 仅做候选筛选与规划决策。
5. **渐进迁移**：Stage 5～15 使用独立测试入口或显式功能开关验证新链路；Stage 16 前后端同步切换新 schema，不建设长期旧响应兼容层。旧实现只保留短期回滚开关，虚构 fallback 在默认和回滚路径中均须移除。
6. **不按 P 编号机械开发**：P 级代表价值，不代表依赖顺序。P0-9 因涉及会话、重规划和 Agent 架构调整，按要求整体放在 **P2 全部完成之后、P3 开始之前**。
7. **PC 优先**：当前阶段不引入移动端和平板适配目标。
8. **同步优先**：景点、天气、酒店、餐饮和路线等外部查询按依赖顺序同步串行执行；本计划不实施查询并发优化。
9. **开发环境边界**：本轮仅面向本地开发环境，复用 `.env` 中现有 LLM 配置；运行拓扑固定为单机、单进程、单实例，不引入 Redis、消息队列、分布式锁或多实例共享状态。
10. **按风险测试**：确定性后端规则和外部数据解析使用轻量 pytest；前端视觉、地图、PDF 和 SSE 可使用构建检查与记录化人工验收，不为追求形式统一建设完整自动化测试平台。

### 1.2 Stage 完成定义（Definition of Done）

每个 Stage 均须满足以下通用条件：

- 确定性后端规则和数据解析通过轻量单元/契约测试；涉及的前端代码通过类型检查和生产构建；
- schema、解析器、TimelineCalculator、BudgetCalculator、TimelineValidator、路线回退和 Coordinator 状态决策不得只依赖人工页面观察；前端视觉、地图、PDF、SSE 等集成能力允许采用有检查清单和结果记录的人工验收；
- API/schema 变更同步更新前端类型和接口文档；
- 外部调用可观测，日志包含 `request_id`、阶段名、耗时、结果状态，不打印 Key 和完整敏感请求；
- 降级、失败和空结果均有显式行为，不使用虚构数据兜底；
- 保留可回滚点，且未破坏上一个 Stage 已通过的验收用例。

### 1.3 Stage 开工与评审清单

每个 Stage 开工前在对应变更说明中补齐以下内容，不要求在本文预写函数级设计：

- 前置 Stage 与 Gate 是否通过；
- 本 Stage 的输入/输出契约及主要修改文件；
- 新增配置及其 Settings 入口；
- 实现步骤和明确不做事项；
- pytest、类型检查、构建或记录化人工验收中的适用项；
- 迁移方式、功能开关和回滚步骤；
- 退出条件对应的验证结果。

配置在首次使用的 Stage 即进入 `Settings`，统一使用：`AMAP_PROVIDER`、`AMAP_API_KEY`、`AMAP_WEB_JS_KEY`、`TRIP_CACHE_MAX_ENTRIES`、`TRIP_CACHE_TTL_POI_SECONDS`、`TRIP_CACHE_TTL_WEATHER_SECONDS`、`TRIP_CACHE_TTL_ROUTE_SECONDS`、`TRIP_REPORT_DIR`、`TRIP_REPORT_MAX_UPLOAD_MB`、`PLAN_TOKEN_SECRET`、`TRIP_REPAIR_MAX_ATTEMPTS`、`TRIP_REPAIR_MAX_SAME_TYPE_ATTEMPTS`。Stage 33 只清理重复读取，不延迟前述配置接入。

---

## 二、目标架构与迁移边界

### 2.1 目标调用链

```text
Trip API
  → 线程池中的 TripCoordinator（同步串行业务编排，不阻塞 FastAPI 事件循环）
    → DestinationResearchAgent（结构化解析额外要求；无法定性则 ignored）
    → AmapService（检索景点、美食级候选与可用天气）
    → DestinationResearchAgent（Stage 18 起结合天气筛选主要点位）
    → ItineraryPlannerAgent（PrimaryItineraryDraft）
    → AmapService + DestinationResearchAgent（连续住宿记录）
    → AmapService + DestinationResearchAgent（仅为未占用餐次按锚点检索/筛选便餐）
    → ItineraryPlannerAgent（完整 ItineraryDraft）
    → RouteService（固定方式；混合方式返回多个真实候选）
    → ItineraryPlannerAgent（仅混合交通时选择真实 route ID）
    → TimelineCalculator / BudgetCalculator（确定性计算）
    → TimelineValidator（真实性与可执行性校验）
    → 校验失败时由 TripCoordinator 替换路线、替换餐厅、重排 PrimaryDraft 或删点并重建受影响下游（有限循环）
  → TripPlan（唯一正式结果）
```

前端只能消费 `TripPlan`，不得消费 Agent 原始输出或 `ItineraryDraft`。

### 2.2 代码目录建议

目录名称可在实现时微调，但职责不得重新混合：

```text
backend/app/
  agents/
    destination_research_agent.py
    itinerary_planner_agent.py
  coordinators/trip_coordinator.py
  models/
    source.py
    constraint.py
    candidate.py
    itinerary.py
    trip.py
  services/
    amap_service.py
    constraint_policy.py
    route_service.py
    timeline_calculator.py
    budget_calculator.py
    timeline_validator.py
    cache_service.py
    report_archive_service.py
  providers/
    amap_provider.py
    mcp_amap_provider.py
    http_amap_provider.py
  api/routes/
    trip.py
    trip_stream.py
    report_history.py
    trip_revision.py
```

### 2.3 迁移期间的硬约束

- `MultiAgentTripPlanner` 仅作为 Stage 16 前及短期回滚用的临时旧入口，不继续堆叠新能力；其虚构 fallback 必须在 Stage 16 前移除；
- 禁止新增 `[TOOL_CALL:...]` Prompt；
- 禁止 Agent 输出或改写地址、坐标、距离、路线耗时、事实价格和预算合计；
- 新链路不得调用 `_create_fallback_plan` 一类虚构兜底；
- Stage 5～15 的新 schema 内部演进优先采用增量字段；必要的破坏性变更在 Stage 16 与当前 Vue 前端同步切换。因无其他调用方，不额外建设长期 API 版本或旧响应迁移适配器；
- 当前结构化 TripPlan 只保存在 Pinia，并用 sessionStorage 做当前标签页临时恢复；不引入结构化行程数据库；
- 只有用户执行“导出行程为 PDF”后产生的 PDF 才进入历史：同一 PDF 既由浏览器下载到本地，也上传到后端固定磁盘目录；历史 PDF 不支持恢复为结构化行程或继续对话；
- PDF 下载能力必须晚于 Stage 22 的多天独立展开能力；导出按钮不依赖页面当前折叠状态，点击后必须自动在导出容器中展开全部天级卡片；
- 酒店和三餐属于必要安排，只能替换、不能删除；`partial` 只能包含完整可执行日，不能返回缺必要项的日计划；
- 首页 `free_text_input` 中可明确结构化的额外要求是最高用户约束优先级；测试阶段不做澄清，无法定性的子句直接 ignored，且不得影响后续规划；
- 任一 accepted 额外要求无法满足时整个请求 failed；必打卡约束不限定日期，但无法唯一匹配或最终未进入行程时不得返回 partial；
- normalized accepted 约束必须随 TripPlan 保存并由修订令牌签名；首版对话修订不得修改，冲突指令不执行且不改变当前行程；
- TimelineCalculator 和 TimelineValidator 只计算与报告问题，只有 TripCoordinator 可以替换路线、替换同级餐厅、重排 PrimaryDraft、删除可选点和触发重算；
- 删除现有手动编辑入口；Stage 14 起行程页保持只读，直到 Stage 32 上线对话式迭代；
- 新链路在 Stage 5～15 仅通过功能开关或 shadow 模式验证，Stage 16 完成错误、状态与虚构 fallback 治理后才切为默认；
- 旧链路的删除必须晚于新链路全量验收，不在中间 Stage 提前清理。
- 当前 Vue 前端是 `/trip/plan` 的唯一调用方；新前端和后端可同步调整 schema，不维持长期旧响应兼容，只保留 Stage 16 切换期的短期回滚开关；shadow 失败不得影响旧链路响应。
- AmapService 是唯一高德数据边界；MCP 和高德官方 HTTP API 通过统一 Provider 契约接入。Stage 1 选定默认 Provider，运行期不因单次失败自动跨 Provider 切换；Provider 切换不是数据降级，不设置 `degraded=true`。
- 阻塞 MCP、HTTP 和 LLM 调用不得直接运行在 FastAPI 事件循环中；单请求内部仍保持同步串行。
- PDF 写入可持久化固定磁盘目录，程序不自动删除历史 PDF，由开发者手动管理目录；系统临时目录不得作为正式报告目录。
- SSE 客户端断开后设置取消标记；当前无法中断的外部调用结束后停止后续规划，不保存 TripPlan，也不启动后台任务继续执行。
- 无状态修订后端只保证 TripPlan、accepted 约束和修订序号的签名完整性，不保证令牌唯一消费或服务端防重放；同页面防并行、重复提交和迟到响应覆盖由前端处理。
- AmapService 对外坐标统一为 GCJ-02；POI、地理编码、路线 polyline、缓存和前端地图不得混用坐标系，Agent 不参与坐标转换。
- API 层生成或接收 `request_id`，并在线程池、Coordinator、Provider、Agent、缓存、修复循环和 SSE 中传播；日志不得记录 Key、plan token、完整 Prompt、完整自由文本或未脱敏上游响应。

---

## 三、阶段总览

本计划共 **43 个 Stage 单元（Stage 1～33，其中 7、9、10、12、31 按子阶段拆分，Stage 13 拆为 13A～13E）**。当前 Demo 已经实际运行确认可用，不另设基线验收 Stage；测试工具在首次需要它的 Stage 内按最小范围引入。每个 Stage 单元的工作量目标为一个可独立评审的小变更集；如果评审时发现仍包含两个高风险问题，应继续拆分，不应合并赶工。

| 阶段 | 主要交付 | 对应优化项 | 依赖 |
|------|----------|------------|------|
| Stage 1 | 高德 MCP/官方 HTTP Provider、配额与串行稳定性验证 | P0-1、P0-6、P1-3 前置 | 无 |
| Stage 2 | 输入、额外要求、来源、状态与错误契约 | P0-4、P1-1、P1-2 前置 | 1 |
| Stage 3 | AmapService POI/地理编码结构化解析 | P0-1 | 1、2 |
| Stage 4 | AmapService 天气/路线结构化解析 | P0-1 | 1、2、3 |
| Stage 5 | 额外要求解析与标准化调用链 shadow 接入 | P0-2、P0-4 | 3、4 |
| Stage 6 | 多偏好完整检索 | P0-3 | 5 |
| Stage 7A | 主要点位候选池 | P0-5、P0-7（一级检索） | 6 |
| Stage 7B | DestinationResearchAgent 主要点位筛选 | P0-4、P0-5、P0-7（一级筛选） | 7A |
| Stage 8 | PrimaryItineraryDraft | P0-4、P0-7（主要点位分天排序） | 7B |
| Stage 9A | 连续住宿与便餐候选/筛选 | P0-5、P0-7（二三级） | 8 |
| Stage 9B | 完整 ItineraryDraft | P0-4、P0-5、P0-7（完整规划） | 9A |
| Stage 10A | RouteService 固定交通基础版 | P0-6（固定方式） | 4、9B |
| Stage 10B | 混合交通真实候选选择 | P0-6（混合方式） | 10A |
| Stage 11 | TimelineCalculator | P0-7、P0-8（计算部分） | 10B |
| Stage 12A | BudgetCalculator | P0-7（预算计算部分） | 11 |
| Stage 12B | TimelineValidator | P0-7（确定性校验部分） | 11、12A |
| Stage 13A | TripCoordinator happy path 与 TripPlan 组装 | P0-7 | 12B |
| Stage 13B | 路线、餐厅、酒店局部修复 | P0-7 | 13A |
| Stage 13C | 明确时间重排与删点重建 | P0-7 | 13B |
| Stage 13D | 全局跨日重排与 partial/failed | P0-7 | 13C |
| Stage 13E | Pinia 当前行程状态基础 | P2-1（前端基础） | 13A |
| Stage 14 | 复合时间轴卡片与移除手动编辑 | P0-8（展示部分） | 13D、13E |
| Stage 15 | 分级错误处理 | P1-2 | 13D、13E |
| Stage 16 | 分级 Fallback、结果状态与新链路切换 | P1-1 | 15 |
| Stage 17 | 进程内请求缓存与外部数据降级 | P1-3 | 1、16 |
| Stage 18 | 天气驱动两次草案规划 | P1-5 | 4、13D |
| Stage 19 | 本地地图 Key 风险治理 | P1-6 | 1 |
| Stage 20 | 地图真实路线与时间轴联动 | P1-4 | 10B、14、19 |
| Stage 21 | 本地开发前端代理统一 | P2-6 | 19 |
| Stage 22 | 多天独立折叠与全部展开 | P2-7 | 14 |
| Stage 23 | 图片批量接口 | P2-4 | 13A、21 |
| Stage 24 | 图片加载优化 | P2-5 | 22、23 |
| Stage 25 | 导出重构、自动全展开与 PDF 下载 | P2-3 | 20、22、24 |
| Stage 26 | 固定磁盘目录 PDF 存储与读取 | P2-1（后端） | 21、25 |
| Stage 27 | 双向导航与 PDF 历史入口 | P2-1（前端集成） | 13E、26 |
| Stage 28 | SSE 同步阶段进度 | P2-2 | 13D、15、21 |
| Stage 29 | PC 显示稳定性优化 | P2-8 | 22、24、25 |
| Stage 30 | 行前须知 | P2-9 | 18、29 |
| Stage 31A | 无状态修订契约与完整性令牌 | P0-9（后端基础） | 全部 P2 |
| Stage 31B | 对话指令结构化解析 | P0-9（意图层） | 31A |
| Stage 31C | 修订重算与校验闭环 | P0-9（编排层） | 31B |
| Stage 32 | 对话式迭代前端与恢复 | P0-9（前端） | 31C |
| Stage 33 | 统一配置管理 | P3-1 | 32 |

> 顺序检查：P2 的最后一个阶段是 Stage 30；P0-9 位于 Stage 31A～32；P3 从 Stage 33 开始，满足“P0-9 放到 P2 之后、P3 之前”。本计划不存在数据查询并发 Stage。
>
> 并行说明：Stage 13E 在 13A 完成后即可与 13B～13D 并行开发；编号位置不表示必须等待 13D。Stage 14 必须同时等待 13D 和 13E。

---

## 四、详细实施阶段

### Stage 1：高德 MCP 与官方 HTTP Provider 技术验证（独立难点 Stage）

**目标**：消除 02 文档明确列出的外部能力不确定性，为后续设计提供实测依据。

**范围**：

- 定义最小 `AmapProvider` 契约：POI 检索、地理编码、天气和路线；MCP 与官方 HTTP Provider 必须能映射到同一原始 DTO 和领域模型；
- 核验目标版本 `amap-mcp-server` 的工具名、入参、返回内容层级和异常形态；同步核验高德官方 HTTP 对等接口的字段、鉴权、错误、限流和配额；
- 核验 POI ID、经纬度、天气日期、路线距离、耗时、polyline 是否稳定可取；
- 核验餐饮 POI 营业时间是否可取、缺失形态、跨天/分段营业表达和可确定性解析范围；无法稳定解析的值按 unavailable 处理，不交给 LLM 推断；
- 分别验证步行、驾车、公交路线；确认公共交通返回能否可靠区分地铁、公交和旅游专线大巴，以及驾车路线用于打车语义时可取得的计费输入；记录地址调用与坐标调用差异；
- 验证同步串行连续调用下 MCP 实例的稳定性、超时行为和资源释放；
- 查询目标 Key 当前官方配额、限流和计费规则；验证 `.env` 中开发配置但不打印或写入任何真实 Key；
- 对典型 1～3 日请求测量调用数、延迟与失败率，产出决策记录。
- 根据字段完整性和连续调用稳定性选定当前默认 Provider；运行期单次失败只进入该 Provider 的重试、缓存或错误路径，不自动切换另一 Provider。若后续人工改配置切换 Provider，不设置 `degraded=true`。
- 在本 Stage 内引入最小 `pytest` 配置，仅用于 Provider 解析 fixture 和后续确定性后端测试；不配置覆盖率门槛、CI 或测试报告平台。

**交付物**：`doc/opt_strategy/decisions/amap_provider_validation.md`、脱敏 fixture、Provider 能力矩阵、最小 pytest 运行命令。

**退出条件**：能回答“默认使用哪个 Provider、返回什么、串行调用是否稳定、失败如何识别、配额如何约束”；至少一个 Provider 满足核心字段要求，否则阻断 Stage 3、4、10A、10B、17，不以猜测继续实现。

### Stage 2：核心领域契约

**目标**：先固定各层边界，再迁移调用链。

**范围**：

- 定义 `DataQuality`、`freshness`、`PlanStatus`、`degraded` 和缺失项结构；
- 定义 `AmapProviderName`、统一 Provider 错误分类和来源元数据；Provider 是技术通道，不改变 `sourced` 语义；
- 将旅行天数限制为 1～3 天、偏好限制为最多 3 个；校验 `travel_days == end_date - start_date + 1`；
- 将交通方式、住宿类型和偏好定义为前后端一致的受控枚举，拒绝未知值；
- 定义 `ExtraConstraint` 封闭判别联合、`ExtraConstraintParseResult` 和 `ignored_extra_requirements`；首版只允许 `must_visit_poi / dietary_exclusion / explicit_time_constraint / mobility_constraint`，每个子句独立输出 `accepted / ignored` 和稳定原因码；
- 固定约束层级为“系统硬约束 → accepted 额外要求 → 三级选址优先级/普通偏好 → Planner 建议”；ignored 约束不允许进入任何后续模型；
- 为四类约束定义执行字段和失败语义：必打卡为 trip scope 且未匹配/未安排即整单 failed；accepted 时间约束同时产生关联的 required 目标；饮食排除使用确定性大众印象规则表，其校验范围不包含 `lodging_side` 早餐；时间约束字段不全则 ignored；移动约束只接收可按路线距离验证的限制，数据源不支持的无障碍要求 ignored；系统默认单次 walking 路线不得超过 2000m，超限只淘汰该方式并触发 taxi/transit 回退；
- 定义结构化候选、`POI_ID / DAY_START / LUNCH_SLOT / DAY_END` 空间锚点、带非餐饮点 `day_part`、美食级餐点 `meal_role` 和 `provisional_meal_slots` 的 `PrimaryItineraryDraft`、连续 `LodgingRecord`、`MealDraft`、完整 `ItineraryDraft`、路线/时间/预算模型和 `TripPlan`；
- 目的地时间固定为 `Asia/Shanghai`，TripPlan 日期时间使用带 `+08:00` offset 的 ISO 8601；营业时间解析明确跨午夜、`24:00`、星期和不可解析值语义；
- AmapService 领域坐标和 route polyline 统一为 GCJ-02；Provider 原始坐标系必须显式记录，若非 GCJ-02 只能在 Provider 层转换。schema/fixture 覆盖经纬度顺序颠倒、范围非法和坐标系不一致；缓存值及其版本包含坐标系语义；
- 路线优先使用数据源原生 ID；数据源无 ID 时按 Provider、规范化起终点、方式、策略和路线摘要生成请求级确定性 `route_id`，只承诺当前候选集合内稳定；
- 金额使用 `Decimal`、币种固定 CNY、保留两位小数；费用项包含单价、数量、计价单位、小计和依据。`unavailable` 不参与“已知费用合计”，不得把该合计描述为完整总预算；
- 住宿集合统一表示为 `lodging[0..lodging_count-1]`：complete 时 `lodging_count=N`，partial Day1..DayK 且 K<N 时为 K+1；它映射返回日的 `start_hotel_id / end_hotel_id`。`PrimaryItineraryDraft` 每日至少包含 1 个景点或美食级主要点，美食级餐点按 `meal_role` 占用对应餐次且不得重复补便餐；完整 `ItineraryDraft` 才加入起止酒店和未占用餐次的便餐，酒店、三餐、必打卡点 required，其他主要点位才允许 `removal_rank`；
- 定义 `BreakfastArrangement` 判别联合：`lodging_side` 不含 POI ID、地址、坐标和路线，默认离店前完成但仍计入三餐齐全，费用无来源时为 unavailable；`gourmet` 必须引用真实餐饮 POI并进入路线、时间轴、营业时间和预算校验；
- 所有建议时长单位固定为分钟。普通景点允许 30～240 分钟，美食级和便餐级在外用餐允许 40～120 分钟；Agent 输出缺失或越界属于非法结构化输出并进入有限重试，Calculator 不猜默认值，修复过程不得压缩到类型下限以下；
- 交通枚举区分 `walking / public_transit / self_drive / mixed`；混合候选内部进一步区分 `walking / taxi / transit`，其中 `transit` 仅按数据源事实细分地铁、公交或旅游专线大巴，`mixed` 不得混入 `self_drive`；
- 将所有请求级交通枚举定义为默认偏好而非硬约束；偏好候选不可用或被系统/accepted 约束过滤时允许回退到其他无需额外前提的真实方式，公共交通偏好不得假设用户具备自驾条件；回退选择键固定为 `(duration_seconds, distance_meters, route_id)`；
- 定义路段级 `MAX_WALKING_SEGMENT_METERS=2000`：超过上限只淘汰 walking 候选并触发 taxi/transit 回退，不直接产生计划失败；
- 定义 partial 只能在完整日期范围经过跨日重排及有限修复后生成，并且是从首日开始的最长连续前缀；前缀截断、住宿记录、预算晚数以及美食偏好下的 1～K 数量规则必须能由 schema 明确表达；
- 预算契约固定携带 2 名成人、1 间房、`lodging_count=len(lodging)` 和各交通方式的计价单位；完整计划为 N，partial Day1..DayK 且 K<N 时为 K+1；
- 定义统一错误响应 `request_id / error_code / message / retryable / details`；请求字段错误映射 422、修订冲突映射 409、上游非法响应映射 502、配置或服务不可用映射 503、超时映射 504，业务不可行使用 200 + `failed`；合法空候选或无可行安排进入 `partial/failed`，核心技术链路重试和缓存降级后仍无可信输入则返回 API 错误。额外要求解析失败是明确例外，按 `parse_failed` 进入 ignored；成功使用 stale 缓存则返回计划并设置 `degraded=true`；
- `TripPlan` 必须携带完整 normalized `accepted_constraints` 及其稳定 ID，不能只保留活动上的 `constraint_ids`；定义 `RevisionConstraintConflict`，稳定错误码为 `revision_constraint_conflict`，并明确冲突响应不包含新版 TripPlan；
- 将可缺失事实字段改为 `Optional`，缺失值为 `null`，不得用 `0` 或空字符串伪装真实值；
- 餐饮营业时间有来源值时支持两阶段校验：美食级/便餐级午晚餐候选在固定餐窗内存在连续 ≥40min 可用区间；美食级早餐因没有固定候选餐窗而跳过预筛。最终阶段统一校验完整覆盖实际用餐区间；缺失时记录 `unavailable` 和核实提示；
- 写 schema 正反例和序列化兼容测试。

**不做**：不接 MCP、不改前端页面。

**退出条件**：1～3 天、日期一致性、最多 3 个偏好、全部枚举、accepted/ignored 额外要求、必要项/删除排名、完整日 partial、预算假设和 02 文档 4.1～4.7 的真实性/来源/状态组合均能由模型表达并通过正反例测试。

### Stage 3：AmapService POI 与地理编码结构化解析

**目标**：让景点、酒店、餐厅和坐标进入统一的真实数据入口。

**范围**：

- 通过 Stage 1 选定的 `AmapProvider` 获取 POI 与地理编码原始响应，并解析为统一 Pydantic 模型；Provider 细节不得泄漏到 Agent 或 API schema；
- 餐饮 POI 必须解析高德来源 ID、具体名称（包含返回的分店名）、地址和坐标；不得用搜索关键词或“川菜馆”等 POI 类别填充名称；
- 按 Stage 1 能力矩阵解析可确定识别的营业时间与适用日期/星期；缺失或无法可靠解析时设为 `null/unavailable`，不得猜测；
- 保留 `source_id`、抓取时间、原始类型和数据质量元数据；
- 输出坐标统一为 GCJ-02；Provider 原始坐标系不同则在 Provider 层转换并保留转换元数据，解析 fixture 覆盖坐标顺序、范围和坐标系错误；
- 对空结果、必需字段缺失、非法坐标、返回格式漂移显式报错或返回空集合；营业时间等可选字段缺失不得误判为必需字段错误；
- 解析逻辑用 Stage 1 fixture 做契约测试。

**不做**：不实现筛选策略、不实现缓存。

**退出条件**：不再返回 TODO 空列表；任何输出 POI 都有合法来源 ID 和坐标；餐饮营业时间的可解析、缺失、分段和非法格式 fixture 均有稳定结果，格式漂移不会静默产生错误实体。

### Stage 4：AmapService 天气与路线结构化解析

**目标**：补齐其余确定性外部事实。

**范围**：

- 天气解析为带日期和来源的结构化数据，超出预报范围标记 `unavailable`；
- 路线解析为距离、耗时、方式、路径坐标和策略元数据；
- 路线失败不得返回空字典冒充成功；
- 建立天气与三类路线 fixture 契约测试。

**退出条件**：AmapService 核心数据调用不再存在未解析 TODO；路线字段可用同起终点和策略复核。

### Stage 5：额外要求解析与标准化数据调用链 shadow 接入

**目标**：移除 Agent Prompt 中脆弱的文本工具调用。

**范围**：

- 新建最小 `TripCoordinator` 骨架，由其直接调用 AmapService；
- 在任何候选检索前，使用 DestinationResearchAgent 的独立 JSON mode、`temperature=0` 调用逐子句解析 `free_text_input`；仅 accepted 约束进入 Coordinator，无法归类、对象不明确或多义的子句直接进入 ignored，不发起澄清；
- 解析响应使用 `extra="forbid"` 和有限重试；重试耗尽时将未确认子句以 `parse_failed` 记入 ignored，并按无额外约束继续，不能导致整次规划失败。该规则仅适用于额外要求解析；PrimaryDraft、ItineraryDraft、RouteSelection 等核心输出解析失败且重试耗尽时返回 API 错误；
- 对 accepted `must_visit_poi` 通过 AmapService 按名称和城市精确检索：先限制城市/adcode，再做规范化名称完全匹配并按来源 ID 去重；分店视为不同实体，多个不同 ID 完全匹配时判为不唯一，模糊匹配不得由 LLM 宣称唯一。只有唯一真实 POI ID 才能进入 required 候选，无法唯一匹配时立即返回 failed；
- 对 accepted `dietary_exclusion` 调用确定性规则表过滤全部系统选择的美食级和便餐级 POI；例如 `avoid_spicy` 直接排除川菜、湘菜、重庆火锅、麻辣烫等，不检查单店的不辣菜品；`lodging_side` 早餐由用户自行选择，不进入该规则校验；
- `explicit_time_constraint` 字段不完整时 ignored；accepted 时为同一目标同时生成关联的 trip-scope `must_visit_poi`，精确匹配为 required 后再执行时间约束，无法唯一匹配或最终未安排时整单 failed；`mobility_constraint` 仅接受可按路线距离校验的限制，不支持的完整无障碍要求 ignored；默认 2000m 上限只约束 walking 方式资格，点位间距超过上限时必须继续查询 taxi/transit，不能仅因此判定日期或整单 failed；
- 景点与天气数据以结构化对象传入后续处理；酒店和便餐仅验证结构化调用接口，实际锚点检索推迟到 Stage 9A；
- 新链路通过独立测试入口、功能开关或 shadow 模式运行，不在本 Stage 替换默认 `/trip/plan`；
- 新链路删除 `[TOOL_CALL:...]` 依赖，旧链路仅作为 Stage 16 前的临时兼容入口，不再扩展能力。

**不做**：不在此 Stage 完成新 Planner、路线闭环或时间轴。

**退出条件**：四类封闭约束均有 accepted/ignored 正反例；明确时间要求同时产生 required 目标；必打卡无法唯一匹配时整单 failed；`avoid_spicy` 固定过滤真实餐饮 POI 且不校验住宿侧早餐；ignored 对下游零影响；新链路不再要求 LLM 生成工具调用字符串，Amap 等外部数据失败能传播为显式错误，默认开发入口仍可回滚。

### Stage 6：多偏好完整检索

**目标**：所有用户偏好都进入候选检索，不再只取第一项。

**范围**：

- 建立偏好到检索关键词的可测试映射；
- 请求层最多接受 3 个偏好；按用户选择顺序同步串行检索并按来源 ID 去重；
- 为每个偏好设置候选上限和合并后的总候选上限，避免 Agent 输入无界增长；
- 保留每个候选命中的偏好标签，供筛选 Agent 使用；
- 无偏好时使用明确的通用景点策略。

**退出条件**：多偏好样本中每个标签都有检索记录；合并结果稳定、可去重、可追溯。

### Stage 7A：主要点位候选池

**目标**：只建立第一选址优先级的景点与美食级餐点候选，避免在活动顺序未知时提前选择酒店和便餐。

**范围**：

- AmapService 按全部偏好和 accepted 必打卡约束同步串行检索景点与美食级餐点；
- `dietary_exclusion` 规则表先过滤美食级餐点候选；
- 美食偏好开启时建立美食级候选池，关闭时不提升餐厅为主要点位；
- 保留来源 ID、命中偏好、具体名称、地址、坐标和数据质量。

**不做**：不检索酒店和便餐，不生成日期或活动顺序。

**退出条件**：主要点位候选全部真实可追溯；必打卡唯一 POI 一定在候选池且标记 required；饮食排除规则命中的候选不会进入池。

### Stage 7B：DestinationResearchAgent 主要点位筛选

**目标**：筛选第一优先级主要点位，并约束 Agent 只能引用已有 ID。

**范围**：

- accepted 约束优先于普通偏好；必打卡 POI 必须保留；
- 美食偏好开启时全程选择 1～N 个美食级候选且每天最多 1 个的最终约束留给 PrimaryDraft 校验；
- JSON mode/`response_format` + `extra="forbid"`，温度 0，最多重试 2 次；
- 输出仅含候选 ID、分类、约束引用和筛选理由。

**退出条件**：未知 ID、ignored 约束引用、必打卡遗漏和 schema 错误均被拒绝；重试上限测试通过。

### Stage 8：PrimaryItineraryDraft

**目标**：第一次调用 ItineraryPlannerAgent，只完成主要点位分天与顺序，为后续住宿和便餐提供稳定锚点。

**范围**：

- 输入筛选后的主要点位、日期、偏好、accepted 约束和可选天气字段；Stage 8 初次交付仅预留并透传天气字段，不得在 Stage 18 上线前用它改变点位、日期或顺序；
- 输出日期、有序主要点位 ID、非餐饮主要点的 `day_part=morning|afternoon`、美食级餐点的 `meal_role=breakfast|lunch|dinner`、`provisional_meal_slots`、用途、`constraint_ids`、`required`、可选点 `removal_rank`、建议时长和理由；美食级餐点的 `day_part` 必须为 null；
- 景点建议时长必须为 30～240 分钟，美食级餐点必须为 40～120 分钟；缺失或越界按非法 Agent 输出重试，不传给 TimelineCalculator 修补；
- 不包含酒店、住宿侧早餐、便餐、路线、最终时刻或费用；
- 必打卡点 required 且不绑定预设日期，由 Planner 在任一可执行日安排；
- 每日至少包含 1 个主要点位；景点和美食级餐点均计数，便餐不计；
- 每个 provisional 餐次槽位标记 `occupied_by_gourmet` 或 `requires_convenience_meal`；美食级餐点按 `meal_role` 占用对应槽位。锚点只能是已有 POI ID 或 `DAY_START / LUNCH_SLOT / DAY_END`：无下午点且午餐尚未选定时必须设置 `dinner_after_anchor=LUNCH_SLOT`，不得误用 `DAY_END`；不得引用 Stage 9A 才会选出的酒店或便餐 ID；
- 美食偏好开启时校验全程 1～N 个且每天最多 1 个美食级餐点；美食级餐点是正式用餐，不得放入其 `meal_role` 对应窗口以外，也不得为同一餐次再生成便餐需求；
- Stage 8 只为美食级餐点分配 `meal_role`，不在路线和实际时刻未知时预判美食级早餐营业时间；该校验推迟到 Stage 11/12B；
- JSON mode/`response_format` + `extra="forbid"`，温度 0.3，最多重试 2 次。

**退出条件**：漏日期、零主要点位、重复点位、必打卡遗漏、错误删除排名、美食级数量越界、`day_part/meal_role` 非法、同餐次重复、`LUNCH_SLOT` 正反例、meal slot 与点位顺序矛盾、ignored 约束引用均有测试；Stage 18 前天气字段对决策零影响。

### Stage 9A：连续住宿与便餐候选/筛选

**目标**：在 PrimaryDraft 已确定后，按二、三级优先级依次完成连续住宿和真实便餐选择。

**范围**：

- AmapService 根据 PrimaryDraft 的空间锚点检索酒店候选，DestinationResearchAgent 选择 N 个 `LodgingRecord`：`lodging[0]` 服务首日出发；中间住宿同时考虑前一日最后主要点位和下一日主要点位；允许连续复用或更换酒店；
- 第 i 天使用 `start_hotel_id=lodging[i]`；非末日 `end_hotel_id=lodging[i+1]`，末日结束酒店为空；
- 酒店确定后，将 `DAY_START / DAY_END` 解析为当日真实起止酒店边界，并严格使用 provisional slots；按午餐优先、晚餐后续的串行顺序处理：先选择缺失午餐，再将 `LUNCH_SLOT` 解析为真实午餐 POI ID，最后建立晚餐搜索锚点；已被美食级餐点占用的餐次不重复检索；
- 未占用午餐位于 `lunch_after_anchor` 与 `lunch_before_anchor` 之间；未占用晚餐位于 `dinner_after_anchor` 之后并在非末日结束酒店之前；美食级或便餐级末日晚餐均作为旅程终点；
- AmapService 返回真实餐饮 POI，先应用 `dietary_exclusion` 规则表，再由 DestinationResearchAgent 选择具体店名/分店；
- 对 Stage 8 已占用午晚餐槽位的美食级餐点及本 Stage 新检索的便餐，营业时间有来源值时只保留在对应固定餐窗内存在连续 ≥40min 可用区间的候选；美食级早餐不做固定餐窗预筛。字段缺失时保留、标记 `unavailable` 并携带核实提示；
- 每日生成一个 `BreakfastArrangement`：未被美食级早餐占用时使用 `lodging_side`，不得伪造 POI；被占用时使用对应 gourmet POI，二者不得并存。

**退出条件**：1～3 日住宿链严格连续；A→B 后次日必从 B 出发；无下午点时 `LUNCH_SLOT` 在午餐选定后解析并成为晚餐真实前锚点；每个午晚餐槽位恰好一个真实 POI；午晚餐营业时间预筛正确，美食级早餐推迟到最终校验；饮食排除正确。

### Stage 9B：完整 ItineraryDraft

**目标**：第二次调用 ItineraryPlannerAgent，将 PrimaryDraft、连续住宿和便餐合并为完整活动顺序。

**范围**：

- 输出每日 `start_hotel_id / end_hotel_id`、有序全部活动、用途、`constraint_ids`、`required`、可选点 `removal_rank`、建议时长和说明；
- 酒店、三餐和必打卡点 required 且无删除排名；午餐/晚餐必须引用 PrimaryDraft 中已占用餐次的美食级 POI 或 Stage 9A 补齐的便餐 POI，并且每个餐次恰好插入一次正式顺序；
- 早餐必须引用唯一 `BreakfastArrangement`；`lodging_side` 计入三餐完整性但不作为外出活动插入路线顺序，`gourmet` 作为正式活动插入对应位置；
- 不得改变 PrimaryDraft 中主要点位的相对顺序和 required 状态；如需改变必须回到 Stage 8 重建；
- 禁止输出地址、坐标、路线、最终时刻和费用；使用 JSON mode、`extra="forbid"`、温度 0.3 和最多 2 次重试。

**退出条件**：住宿连续、三餐齐全且无重复餐次、美食级/便餐均进入对应顺序、必打卡保留、未知 ID、非法字段和 PrimaryDraft 漂移测试全部通过。

### Stage 10A：RouteService 固定交通基础版（独立难点 Stage）

**目标**：提供统一、可追溯的相邻点位真实路段。

**范围**：

- 实现 `TripCoordinator → RouteService → AmapService` 调用；
- RouteService 先查询请求偏好方式；该方式不可用、walking 超过 2000m 或被 accepted 移动约束过滤时，自动同步补查其他无需额外前提的真实方式。walking 回退集合为 taxi/transit；public_transit 回退集合为合格 walking/taxi，不假设用户有车；self_drive 回退集合为合格 walking/taxi/transit。固定方式回退由 Coordinator 按 `(duration_seconds, distance_meters, route_id)` 最小值确定性选择；
- 输入只接受已有点位坐标，输出距离、耗时、方式、polyline、来源、策略；transit 还应保留数据源提供的各步行接驳段距离；
- 公交初次查询使用 `day_part`、餐次槽位和活动顺序推导的预计出发时刻，并记录查询日期与时刻；TimelineCalculator 生成实际时刻后若偏差超过 30 分钟，允许最多重查一次公交路线并重算时间轴；
- 对 `start_hotel_id`→首点、点位间逐段计算；非末日还计算末点→`end_hotel_id`，末日以晚餐为最后活动且不生成返店段；
- 任一必要路线缺失时返回可识别的校验输入，不生成直线兜底。

**退出条件**：典型单日所有相邻段可追溯；1999m walking 可选、2001m walking 被淘汰并成功回退 taxi/transit；public_transit/self_drive 偏好不可用时按定义集合回退且结果稳定；只有所有合格替代路线也缺失时才返回必要路线失败。

### Stage 10B：混合交通真实候选选择（独立难点 Stage）

**目标**：在不生成路线事实的前提下，为“混合交通”逐路段选择更方便的真实路线。

**范围**：

- RouteService 对每个相邻路段同步串行查询高德实际支持的步行、公共交通和驾车路线；驾车路线在混合语义中标记为 `taxi`，公共交通按返回事实细分地铁、公交或旅游专线大巴；
- ConstraintPolicy 在 RouteSelection 前过滤：walking 总距离 >2000m 的候选删除；transit 中任一实际步行接驳段超过系统上限或 accepted 更严格上限时删除；taxi 候选不因点位距离 >2km 被删除；
- 候选统一包含 route ID、业务方式、底层策略、距离、耗时、换乘信息、polyline 和来源；不可用方式不进入候选集合，数据源未明确支持旅游专线时不得生成该候选；
- ItineraryPlannerAgent 使用独立的 `RouteSelection` JSON mode、`temperature=0.3` 调用，只能返回已有 route ID 和选择理由；
- `RouteSelection` 使用 `extra="forbid"` 校验、候选 ID 校验和最多 2 次有限重试；
- Agent 无有效选择时由 Coordinator 从合格候选中按最短耗时确定性选择；步行被过滤但仍有 taxi/transit 时必须继续规划，仅在全部真实替代候选为空时进入必要路线失败。

**退出条件**：每段选择均引用过滤后的真实 route ID；3km 点位不会选择 walking，也不会仅因超出 2km 而 failed；transit 步行接驳、非法 ID、全候选缺失和重试耗尽测试通过。

### Stage 11：TimelineCalculator

**目标**：用确定性代码生成符合 02 文档约束的完整时间轴。

**范围**：

- 使用 `Asia/Shanghai`、分钟精度的纯函数前向排程；严格保持 ItineraryDraft 活动顺序，Calculator 不移动、替换或删除活动；
- 生成出发、交通、独立缓冲、到达、活动/用餐、离开和非末日抵达结束酒店事件；末日晚餐结束事件即为旅程终点；
- 每段真实交通外强制加入至少 30 分钟独立 `buffer`；
- 为等待固定餐窗或营业开始生成独立 `wait` 事件；`wait` 不得替代、吞并或延长强制 `buffer`；
- 所有在外用餐至少 40 分钟；
- 午餐必须完整落在 12:00—14:00，普通日晚餐完整落在 18:00—20:00，末日晚餐完整落在 17:00—18:00；
- 非餐饮主要点中，`day_part=morning` 必须在实际午餐开始前结束，`afternoon` 必须在实际午餐结束后开始；点位不得与午餐重叠。美食级餐点忽略 `day_part`，严格进入其 `meal_role` 对应的早餐、午餐或晚餐位置和窗口；
- 住宿侧早餐 09:00 出发，美食级早餐 08:00 出发；住宿侧早餐不是已核实 POI，不生成店名、地址或费用；
- `lodging_side` 只生成离店前已完成的早餐安排说明，不生成外出 Activity、RouteSegment、wait 或 buffer；`gourmet` 早餐生成实际外出活动及相邻路线；
- TimelineCalculator 生成美食级早餐的实际开始/结束时刻；营业时间校验由 Validator 执行，Calculator 不自行替换餐厅；
- 非末日最晚 21:00 抵达 `end_hotel_id`；末日无结束酒店，晚餐须在 18:00 前结束；单日游按末日规则；
- 日期类型始终以 `day.date == request.end_date` 判定，不以“当前返回数组最后一天”判定；因此 partial Day1..DayK 且 K<N 时，DayK 仍按非末日规则计算；
- 公交实际出发时刻与查询参考时刻偏差超过 30 分钟时返回 `transit_time_refresh_required`，由 Coordinator 最多重查一次该路段后重新调用 Calculator；
- 超窗时返回 `TimelineCalculationResult`，包含冲突区间和按 `removal_rank` 排序的可选修复候选；问题按日期、活动序号、问题优先级稳定排序，不得直接修改草案、替换餐厅或删除点位。

**不做**：不实现前端时间轴。

**退出条件**：四种出发/结束窗口、三种用餐窗口、上午/下午边界、首末日、住宿连续均有测试；点位跨午餐会返回结构化问题；超窗仅产生报告，输入草案保持不变。

### Stage 12A：BudgetCalculator

**目标**：让所有费用明细与汇总只由确定性代码产生。

**范围**：

- BudgetCalculator 成为单项预估费用和预算汇总的唯一提供方；
- 全部金额使用 `Decimal`，币种固定 CNY，按两位小数输出；费用项包含单价、数量、计价单位、小计、数据质量和估算依据；
- 固定输出预算假设：2 名成人、1 间房、实际 `lodging_count` 晚；门票、餐饮和公共交通按 2 人计算，酒店按每晚 1 间房计算，打车按 1 车计算；
- `lodging_count` 必须直接取输入住宿记录数量：完整 N 天为 N，partial Day1..DayK 且 K<N 时为 K+1；单日完整行程为 1，末日晚餐结束后不增加住宿费；
- 住宿侧早餐只有在来源明确证明含早或有价格时才计入酒店费/餐饮费，否则该项为 `unavailable`，不得默认生成双人早餐费用；
- 打车费用使用对应驾车路线距离/耗时和城市可配置的明确计价规则；自驾仅在存在明确里程、停车等规则时估算，缺少规则则为 `unavailable`；
- 仅使用事实价格或有明确依据的规则估算，否则为 `unavailable`；
- `unavailable` 项不参与 `known_cost_total`，页面和 API 均将其称为“已知费用合计”，不得声称是完整总预算；
- 记录每个估算项的依据和 `estimated_fields`，不允许 Agent 费用进入正式结果。

**退出条件**：1 日完整计划、3 日完整计划、Day1 partial/Day1..2 partial 的 `lodging_count`，以及 2 人门票/餐饮/公交、1 车打车、预算假设展示数据和总计恒等测试通过；事实价格、规则估算、无规则三类测试通过。

### Stage 12B：TimelineValidator

**目标**：建立正式结果组装前的真实性与可执行性质量闸门。

**范围**：

- 校验未知 POI、非法/修复失败坐标、缺失路线和 Agent 越界字段；
- 校验时间冲突、餐饮时长、独立缓冲、每日窗口和首末日边界；
- 校验固定午晚餐窗口、day_part 与最终时刻一致性、非餐饮主要点不跨午餐；`meal_role=lunch` 的美食级餐点作为午餐本身，不执行非餐饮点跨午餐校验；
- 校验每日至少 1 个主要点位；景点和美食级餐点计数，便餐不计；
- 校验实际住宿记录数等于 `lodging_count`：complete 为 N，partial Day1..DayK 为 K+1；住宿与相邻返回日期严格连续，酒店与三餐必要项齐全。仅当返回日 `date == request.end_date` 时校验末日晚餐 17:00—18:00、无结束酒店和无返店段；partial K<N 时所有返回日均按非末日规则，DayK 必须抵达 `lodging[K]`；
- 校验午餐和晚餐均引用本次高德候选中的具体美食级或便餐级 POI，并实际出现在活动顺序、相邻路线和时间轴中；类别名称、仅有页面文案或同餐次重复安排均视为必要餐食校验失败；
- 餐馆营业时间有来源值时，统一校验其完整覆盖最终早餐/午餐/晚餐实际用餐区间；美食级早餐没有候选餐窗预筛。冲突时返回同级餐厅替换问题；营业时间缺失时允许通过，但保留 `unavailable` 和核实提示；
- 校验每条 accepted 额外要求均可追溯：必打卡及明确时间约束隐含的 required 目标真实存在且未进入删除列表，`dietary_exclusion` 未被系统选择的真实餐饮 POI 违反但不检查住宿侧早餐，时间/移动约束通过确定性检查；ignored 子句没有规划效果；
- 校验选中的 walking 路线 ≤2000m，transit 各步行接驳段满足适用上限；发现超限时返回 `route_mode_ineligible` 供 Coordinator 换路线，不直接把日期判为 failed；
- 校验失败返回结构化问题列表，不允许作为 `complete` 返回；
- 标记首个无法形成完整日的日期，供 Coordinator 截取从首日开始的最长连续前缀；任一 accepted 约束无法满足时直接整单 failed，不进入 partial 分支。

**退出条件**：02 文档 4.5～4.7 的验收规则形成自动化测试，且每种失败均有稳定问题码。

### Stage 13A：TripCoordinator happy path 与最终组装

**目标**：先完成无修复情况下的固定调用链和唯一 `TripPlan` 组装入口。

**范围**：编排 Stage 7A～12B，按来源 ID 回填事实字段；Calculator/Validator 保持无副作用；只有 Validator 无问题时组装 `complete` TripPlan，并回填 normalized `accepted_constraints`、状态、来源和已知费用合计。Coordinator 作为同步函数在线程池执行，不阻塞 FastAPI 事件循环。

**不做**：不在本 Stage 修复失败草案，不生成 partial。

**退出条件**：1～3 日 happy path 可从 fake Provider/LLM 生成完整 TripPlan；未知 ID、缺路线或 Validator 问题不会被错误组装为 complete。

### Stage 13B：路线、餐厅与酒店局部修复

**目标**：对不改变主要点位集合的局部问题实施确定性修复。

**范围**：

- `route_mode_ineligible` 先替换过滤后的真实路线；公交参考时刻偏差超过 30 分钟时最多重查一次并重算时间轴；
- 餐饮锚点、营业时间或用餐窗口冲突时只替换同级餐厅。候选先通过饮食与营业时间过滤，再按“前锚点→餐厅→后锚点真实路线总耗时、总距离、POI ID”排序；末日晚餐只计算前一段，禁止使用直线距离；
- 酒店造成必要路线缺失或时间轴不可行时替换同住宿类型酒店；酒店变化使前一日结束边界、下一日开始边界及关联餐次、路线、时间轴和预算全部失效并重建；
- 引入 `RepairContext`：全局最多 8 次修复、同类问题最多 3 次，记录 `request_id`、`attempt_id`、已尝试 route/meal/hotel ID、草案指纹和诊断日志；指纹重复立即停止，配置值可根据 Stage 1 实测下调；
- 问题优先级固定为必要路线/方式 → 餐厅 → 酒店；accepted 约束始终不变。

**退出条件**：覆盖 3km 步行回退、公交时刻刷新、同级餐厅两段真实路线比较、酒店替换及相邻日期失效；重复候选和重复草案不会形成循环。

### Stage 13C：明确时间重排与删点重建

**目标**：处理需要改变 PrimaryDraft 的问题。

**范围**：`explicit_time_violation` 回到 Stage 8 调整日期、顺序、`day_part` 或 `meal_role` 位置；普通超窗最后才按 `removal_rank` 删除可选主要点。任何变化都从 PrimaryDraft 重建受影响住宿、餐次、完整草案、路线、时间轴和预算；酒店、三餐、必打卡及其他 accepted required 目标不得删除；删除美食级点后仍须满足数量下限并补齐对应餐次。

**退出条件**：明确时间重排和删点均触发正确失效范围；required 项不可删除；真实交通、最低用餐时间和 buffer 不会被压缩。

### Stage 13D：全局跨日重排与 partial/failed

**目标**：在完整计划不可行时完成最后一次全局修复，并严格构造 partial 或 failed。

**范围**：

- partial 前必须在完整 N 天范围回到 Stage 8，跨日重排可选主要点、美食级餐点和餐次，并重建全部下游；
- 全局修复仍失败后，从最大 `K=N-1` 开始重新构建 Day1..K，而不是直接截取数组；K<N 时重新确认或选择 `lodging[K]`，DayK 按非末日规则抵达该酒店，重查路线并重算时间轴、K+1 晚住宿和预算；
- 每个 K 都必须重新通过完整 Validator；失败则递减 K，K=0 时 failed；不得跳过失败日返回后续日期；
- 美食偏好前缀须含 1～K 个美食级餐点且每天最多 1 个；任一 accepted 约束无法满足时直接 failed，不进入 partial；
- 根据结果生成 `complete / partial / failed`，与 `degraded` 独立；保存完整修复诊断但不向用户暴露内部堆栈。

**退出条件**：Day2/Day3 失败、前缀逐级缩短、DayK 结束酒店、K+1 晚预算、美食数量和 accepted 约束失败均有轻量测试。

### Stage 13E：Pinia 当前行程状态基础

**目标**：在新 TripPlan 页面能力开发前建立唯一前端运行时状态入口。

**范围**：安装 Pinia；管理当前 TripPlan、加载/错误状态和原子替换；sessionStorage 只做当前标签页临时恢复；不包含手动编辑、PDF 历史或对话 UI。

**退出条件**：类型检查和生产构建通过；人工验证生成结果、刷新恢复、无当前行程和清空当前会话四种状态，组件不得各自维护正式 TripPlan 副本。

### Stage 14：复合时间轴卡片与移除手动编辑

**目标**：按 02 文档第六章展示可执行时间轴，同时保留现有点位信息密度。

**范围**：

- 左侧时间轨道 + 右侧点位卡片；
- 展示出发/到达/离开、交通、距离、耗时、独立缓冲、用餐和当日行程时长；
- Day 头部展示 `start_hotel` 与 `end_hotel`；酒店不同时明确为换酒店，末日结束酒店为空；
- 景点和美食级餐厅使用同级卡片，但美食级餐厅仍按 `meal_role` 作为正式早餐/午餐/晚餐进入对应时间窗口；同餐次不得再展示便餐。便餐级午餐/晚餐作为时间轴活动展示，并在 Day 底部餐饮条汇总；
- 便餐级午餐/晚餐显示高德返回的具体店名/分店名、地址等来源字段，并在时间轴中作为正式用餐活动展示；Day 底部餐饮条仅做汇总，不得成为便餐的唯一展示位置；
- 未安排美食级早餐时显示“酒店餐厅或附近早餐店”及酒店政策核实提示；非末日显示最后活动到结束酒店的路线，末日时间轴以晚餐结束；
- 预算区域展示 2 名成人、1 间房、实际 `lodging_count` 晚以及公共交通按人、打车按车的计算假设；完整计划显示 N，partial 显示 K+1；
- 预算汇总明确标注“已知费用合计”；unavailable 项单独列出，不把已知合计描述为完整总预算；
- 事实缺失显示“暂无数据”，不使用 `0` 伪装；
- 餐馆营业时间缺失时显示“营业时间待核实”，不得隐藏提示或生成虚假营业时段；
- 删除地址、时长、顺序和删点等所有前端手动编辑入口；对话式迭代上线前行程页保持只读；
- 本 Stage 只预留地图联动事件，真实地图联动由 Stage 20 实现。

**退出条件**：02 文档 6.3 中除地图联动外的验收项通过；代码中不存在直接修改 TripPlan 事实、路线、时间和预算字段的 UI 入口；PC 主流宽度下无信息截断和布局退化。

### Stage 15：分级错误处理

**目标**：让调用方和用户能区分可恢复与不可恢复故障。

**范围**：

- 映射配置错误、网络超时、上游限流、当前 AmapProvider 异常、核心 Agent/草案解析异常和业务不可行；额外要求解析重试耗尽固定映射为 `parse_failed` ignored，不进入本 Stage 的 API 错误分支；
- 上游返回合法空候选、必要 POI/路线确实不存在或有限修复后时间轴仍不可行，属于业务不可行，由 Coordinator 按完整日规则生成 `partial/failed`；
- 技术调用在有限重试且无 live/cached/stale 可用输入后仍失败时返回稳定 API 错误，不构造 `TripPlan`，也不使用 `plan_status=failed` 掩盖技术故障；本 Stage 用 fake cache source 验证 stale 契约，真实缓存与 stale 集成留到 Stage 17；
- API 返回稳定错误码、可展示消息和 `request_id`；
- 前端按类型显示重试、检查配置或稍后再试，不暴露内部堆栈。

**退出条件**：每类错误均有轻量 API 测试和对应 UI 人工验收；合法空结果、技术超时无缓存和 fake stale 三类分流通过，500 不再承载所有语义。

### Stage 16：分级 Fallback、结果状态与新链路切换

**目标**：彻底移除虚构兜底，正确表达完整性与降级。

**范围**：

- 坐标仅允许高德地理编码或带来源缓存修复；
- 删除北京默认坐标、算法平移、虚构名称和 LLM 补事实路径；
- 正确处理 `complete / partial / failed × degraded` 组合；
- partial 只展示从首日开始的最长连续完整前缀，并明确首个失败日及后续截断日期；首日失败时返回 failed；
- accepted 额外要求无法满足时整单 failed，不得通过剔除相关日期返回 partial；
- 前端不得把 partial/failed 显示为完整成功。
- Gate B 全部通过后，将新 TripCoordinator 链路切为默认 `/trip/plan`；当前 Vue 前端同步切换新 schema。没有其他调用方，不建设长期旧响应适配层；旧链路只保留短期功能开关用于回滚；
- shadow 使用独立测试入口或显式开关，不在普通用户请求中双跑外部调用；shadow 失败不影响旧链路响应；切换后默认请求不得再进入旧 Planner 的虚构 fallback，短期回滚路径也必须使用已经移除虚构 fallback 的旧实现版本。

**退出条件**：注入空 POI、坐标失败、路线失败和 Agent 非法 ID 时均不会生成虚构事实；默认开发入口使用新链路，并完成一次可验证回滚演练。

### Stage 17：进程内请求缓存与外部数据降级

**目标**：减少重复调用，并在实时服务失败时提供可追溯降级。

**范围**：

- 使用单进程内有界 TTL 缓存，不引入 Redis、不要求进程重启后保留；根据 Stage 1 实测确定各数据类型 TTL、容量和淘汰策略并补充 ADR；
- 缓存键包含领域操作、标准化参数、数据版本和策略；Provider 元数据随缓存值保存，切换 Provider 后不得误复用格式或语义不兼容的数据；
- 区分 `live / cached / stale`；仅在实时失败时允许 stale；
- 缓存故障不得破坏实时主链路。

**退出条件**：有效命中、过期重取、实时失败转 stale、无 stale 失败四类同步串行测试通过。

### Stage 18：天气驱动两次草案规划

**目标**：天气从展示信息升级为规划输入。

**范围**：

- 启用 Stage 8 已预留但此前不得参与决策的天气字段；Stage 18 上线前后使用同一 schema，避免迁移破坏；
- 仅使用数据源预报范围内的天气，并为 Stage 7A 候选补充室内/室外标签：高德明确提供时标记 `sourced`，否则由受控规则或 Agent 形成 `estimated`，允许 `unknown`；unknown 不得被天气规则武断当作室内或室外；
- 天气先进入 Stage 7B：DestinationResearchAgent 只能从现有真实候选 ID 中重新筛选或替换主要点位；随后 Stage 8 基于更新后的主要点位集合调整日期和顺序，并在理由中引用天气来源；
- 天气导致主要点位集合、日期或顺序变化后，必须从 PrimaryDraft 起重建连续住宿、缺失餐次便餐、完整草案、路线、时间轴、预算和校验结果；
- 无预报时只标记 unavailable，不用气候常识冒充预报。

**退出条件**：恶劣天气触发室外点替换、仅调整日期/顺序、有预报但无需调整、超预报范围和天气查询失败用例均符合真实性边界；未知候选 ID 被拒绝，点位变化后全部下游失效并重建。

### Stage 19：本地地图 Key 风险治理

**目标**：在地图增强前降低本地开发 Web JS Key 被滥用的风险。

**范围**：

- 在高德控制台仅允许实际使用的 localhost/127.0.0.1 开发来源，配置合理配额和最小权限，不使用任意来源通配符；
- 区分前端 Web JS Key 与后端服务 Key，禁止混用；
- 更新 `.env.example` 和本地启动检查清单，日志/文档不出现真实 Key；
- 增加启动时缺失配置检查。

**退出条件**：非本地白名单来源不可使用前端 Key；后端 Key 不进入前端构建产物；控制台配置留人工验收记录。

### Stage 20：地图真实路线与时间轴联动

**目标**：地图只绘制 RouteService 路径，并与时间轴形成双向对应。

**范围**：

- 封装 `TripMap.vue`；
- 分天路线颜色、`start_hotel / end_hotel` 和天序号标记；酒店更换时显示末段到新酒店及次日从同一酒店出发；
- 删除点位直线连接；必要 polyline 缺失时显示不可用状态；
- 时间轴 hover/click 高亮地图段，地图段 hover/click 定位时间轴；
- 使用稳定的 `day_id + segment_id` 建立关联。
- 验证当前 html2canvas 能否稳定捕获高德 WebGL/Canvas 和跨域瓦片；若不能，导出容器使用独立 SVG 路线快照：按 RouteSegment 真实 polyline 绘制酒店、点位、天序号和交通方式，禁止改为点位直线。底图无法捕获时可使用明确标注的无底图真实路线示意；产出导出地图能力记录供 Stage 25 使用。

**退出条件**：地图上的每段路线均可映射回 RouteSegment；02 文档 6.3 的可联动、可追溯要求通过；导出技术记录证明可捕获地图或可生成基于真实 polyline 的 SVG 快照，禁止以直线或空白占位作为成功方案。

### Stage 21：前端代理统一

**目标**：在新增 SSE、PDF 和批量图片 API 前统一所有前端业务请求入口。

**范围**：统一 trip、map、poi、image、stream 和 report 请求配置；当前只定义 Vite 本地开发代理，不设计生产 base URL；删除绕过代理的硬编码地址，并为 JSON、流式响应和 multipart 上传保留独立请求配置。

**退出条件**：代码扫描无业务 API 硬编码 host；本地开发代理和生产构建命令通过冒烟测试（构建检查不代表设计生产部署配置）。

### Stage 22：多天独立折叠与全部展开

**目标**：各天天级卡片互不干扰，并确保任意 1～3 天可以同时完整展开。

**范围**：默认全部折叠，支持单独展开、全部展开和全部折叠，以稳定 day ID 保存展开状态；不改页面整体布局。

**退出条件**：1、2、3 天行程均可全部展开；展开/折叠任一天不影响其他天；当前 TripPlan 更新时无旧状态串扰。

### Stage 23：图片批量接口

**目标**：将 N 个前端图片 HTTP 请求收敛为单次批量请求。

**范围**：

- 后端接受去重后的点位 ID/关键词列表并同步处理，批量返回映射；
- 设置单批上限、超时和单项错误，不因一张失败导致整批失败；
- 服务端复用缓存；前端按一次行程一次或分块请求。

**退出条件**：典型报告的浏览器图片 API 请求数显著收敛；部分失败仍能渲染其他图片。

### Stage 24：图片加载优化

**目标**：保证天级卡片全部展开及导出准备时，图片加载状态准确可判断。

**范围**：图片懒加载、固定尺寸骨架屏、轻量占位图、有限次数重试和最终 fallback；提供导出流程可等待的 `imagesReady` 状态，重试耗尽也必须以明确 fallback 结束等待。

**退出条件**：慢网、404、超时和重试耗尽场景可预测；全部展开后图片均进入成功或最终 fallback 状态，不存在导出流程无限等待。

### Stage 25：导出重构、自动全展开与 PDF 下载（独立难点 Stage）

**目标**：在 Stage 22 已支持全部天级卡片展开的基础上，实现不受当前折叠状态限制的 PDF 下载，并证明完整行程页只生成一个准确 PDF Blob。

**范围**：

- 抽取共享的导出页面准备、截图、分页、文件命名和错误恢复逻辑，消除图片/PDF 重复代码；
- 导出按钮始终可点击，不要求用户预先展开任何天；点击后在独立导出容器中自动展开全部 1～3 天天级卡片，并等待字体、图片成功/fallback 和地图渲染完成；
- 地图必须包含酒店、点位和真实路线信息；地图截图失败必须显式报错或使用明确占位，不得静默导出空白地图；
- 优先复用 Stage 20 已验证的地图捕获方式；捕获不可靠时必须使用 Stage 20 的 SVG 真实路线快照。占位图仅代表本次导出失败，不能满足本 Stage 成功退出条件；
- 实现卡片感知的 A4 跨页处理，避免标题、时间轴事件、点位卡片和图片被页边界任意截断；
- 生成且只生成一次 `Blob` 并触发浏览器下载；导出完成或失败后恢复用户原有展开状态。本 Stage 不接后端保存，“下载并加入历史”中的上传部分由 Stage 27 接入。

**退出条件**：分别从全部折叠、部分展开和全部展开状态点击导出，1、2、3 天 PDF 均包含完整图片、地图、时间轴和餐饮信息；跨页抽查无关键卡片截断、重复或遗漏；导出后页面状态恢复。未通过时阻断 Stage 26、27。

### Stage 26：固定磁盘目录 PDF 存储与读取

**目标**：在 PDF Blob 已验证准确后，接收同一 PDF 并保存到固定磁盘目录。

**范围**：

- 后端提供 PDF 上传、历史列表和下载接口；
- PDF 写入配置项指定的可持久化固定磁盘目录，开发环境默认使用项目下的 `data/trip_reports/`，该目录不提交 Git且不得指向系统临时目录；
- 历史列表请求每次扫描固定目录中的全部 `.pdf` 文件，并根据安全文件名和文件修改时间生成展示信息，不依赖数据库；
- 文件名包含城市、行程日期、导出时间和防冲突短 ID；同名文件不得静默覆盖；
- 校验 PDF 文件头、MIME、大小上限和安全文件名，下载接口防止路径穿越。
- 最终文件名和路径完全由服务端生成，不信任客户端文件名；并发上传使用不同短 ID，在固定目录内以独占临时文件写入，校验成功后原子重命名，同名不覆盖；失败时清理临时文件，历史扫描忽略临时后缀。
- 程序不自动删除或轮转历史 PDF，目录容量由开发者手动管理；失败不得留下可被历史列表识别的半文件。

**不做**：不保存或恢复结构化 TripPlan，不支持从历史 PDF 继续对话；维持当前本地/单用户目录方案，多用户隔离留待后续。

**退出条件**：服务重启后目录内全部合法 PDF 仍可列出和下载；上传、列表、下载、非法文件和路径穿越测试通过。

### Stage 27：双向导航与 PDF 历史入口

**目标**：复用 Stage 13E 的当前行程状态，为 Stage 25 已验证且已下载的同一 PDF Blob 接入固定目录保存和历史入口。

**范围**：

- 行程页始终可返回首页；首页仅在存在有效当前 TripPlan 时显示“返回当前行程”，否则直接访问行程页返回首页；
- 用户点击“导出行程为 PDF”后复用 Stage 25 的单一 Blob：自动全展开导出并触发浏览器下载，再上传完全相同的 Blob 到固定目录；不得二次截图或重新生成 PDF；
- 历史列表读取固定目录中的全部 PDF，仅展示名称、文件时间和下载入口；
- 本地下载成功但上传失败时提示“已下载到本地，但未加入历史”。

**退出条件**：初次进入、双向跳转、刷新恢复、关闭会话、导出/未导出、下载成功但上传失败、重新打开网页历史下载场景通过；历史 PDF 不能作为可编辑行程打开。

### Stage 28：SSE 同步阶段进度

**目标**：在保持后端处理同步串行的前提下，用真实阶段事件替换 `setInterval` 假进度。

**范围**：

- 新增 `POST /trip/plan/stream`，返回 `text/event-stream`；前端使用 `fetch + ReadableStream`，不使用只支持 GET 的原生 EventSource；
- 事件统一包含 `request_id / sequence / stage / status / payload`；最终 `result` 事件携带唯一 TripPlan，`error` 事件携带 Stage 2 的统一错误结构，心跳不计入业务阶段；
- 每个请求建立有界线程安全事件队列和 cancellation token；线程池中的 Coordinator 向队列写事件，异步响应生成器读取并发送。队列写入必须有超时或合并非关键进度策略，不得因客户端消费缓慢无限阻塞外部调用；
- 事件顺序与真实调用链一致：额外要求解析 → 主要点位/天气检索与筛选 → PrimaryDraft → 连续住宿 → 缺失餐次便餐检索/筛选 → 完整 ItineraryDraft → 路线计算/混合选择 → 时间轴与预算 → 校验/全局修复循环 → 完成或失败；Stage 18 前天气只透传、不产生决策事件；
- Coordinator 仍在线程池中同步串行执行，不启动并行查询、并发 Agent 或后台任务；
- 客户端断开后设置线程安全 token；Coordinator 在每次外部调用返回后及下一 Stage 开始前检查，停止后续规划且不保存 TripPlan；`result/error` 只能入队一次，请求结束后释放队列、token 和 worker 引用；Coordinator 只发真实阶段状态，不伪造百分比；
- 前端展示阶段状态并移除假进度定时器。

**退出条件**：正常、修复循环、阶段失败、队列背压和 result/error 唯一性通过接口级轻量测试；客户端断连通过记录化人工验收，确认当前调用结束后无后续阶段、无保存结果且无重复正式结果，队列和 worker 引用被释放。

### Stage 29：PC 显示稳定性与可读性

**目标**：在既定 PC 范围内完成局部视觉收敛。

**范围**：验证目标桌面宽度下时间轴、地图、长名称、缺失字段、partial/degraded 提示、全部天展开和 PDF 历史入口；只做局部 CSS 和信息可读性修正。

**不做**：不做移动端/平板适配，不整体重构页面布局。

**退出条件**：约定的 PC 浏览器与宽度矩阵人工验收通过，无横向溢出和关键状态遮挡。

### Stage 30：行前须知

**目标**：增加旅途级单次展示的预约、穿着和天气提示。

**范围**：

- ItineraryPlannerAgent 通过独立的受限结构化输出生成建议性须知并标记 `generated`；
- 天气提示只能引用可用预报，不将气候建议写成确定天气；
- 页面底部单次展示，不在每天重复；
- 预约事实无来源时使用条件式建议，不编造开放时间或预约政策。

**退出条件**：三类信息均有明确真实性标签；远期天气和缺失预约数据不会生成确定性事实。

### Stage 31A：无状态修订契约与完整性令牌（P2 后、P3 前）

**目标**：在不保存结构化行程数据库的前提下，建立可信的当前行程修订输入输出契约。

**范围**：

- 定义 `TripRevisionRequest`：携带完整 normalized `accepted_constraints` 的当前 `TripPlan`、自然语言指令、当前 `revision_no` 和 `plan_token`；
- 生成与校验签名完整性令牌，令牌绑定 TripPlan、完整 accepted 约束集合及修订序号；客户端删除、替换或放宽 accepted 约束均导致验签失败；
- 签名输入使用稳定规范化 JSON，明确字段排序、时间/Decimal 序列化和令牌版本；签名密钥从现有 Settings 读取且不进入前端；
- 后端不保存 TripPlan、对话消息或结构化版本历史；每次请求以客户端提交且验签通过的当前 TripPlan 为基准；
- 响应返回完整新版 TripPlan、新修订序号和新令牌；失败只返回错误，不修改客户端当前行程；
- 无状态后端只校验提交令牌的完整性，不记录令牌是否已使用，也不声称能拒绝两个并发使用同一合法 token 的请求；同一页面禁止并行、重复提交和迟到响应覆盖由前端依据 base revision、request ID 和当前 revision 处理。

**不做**：不解析自然语言，不执行重规划，不保存服务端版本，不开发聊天 UI。

**退出条件**：合法令牌、篡改 TripPlan、篡改 accepted 约束、规范化序列化和失败保留当前行程测试通过；前端防并行与迟到响应覆盖在 Stage 32 验收，不要求无状态服务端防重放。

### Stage 31B：对话指令结构化解析（独立难点 Stage）

**目标**：将自然语言调整请求约束为有限、可验证的 revision intent。

**范围**：

- 第一版支持独立 intent：增删/替换普通点位、调整日期或顺序、更换酒店、`replace_gourmet_meal`、`replace_convenience_meal`、`change_meal_role`、修改交通偏好；
- 首版每次只允许一个独立修改动作；检测到多个动作时不拆分、不部分执行，返回结构化澄清状态，要求用户逐条提交；
- 定义各 intent 的 Pydantic 判别联合和字段校验；
- 使用 `temperature=0`；Agent 只解析意图、当前实体引用或新实体结构化 query，不直接生成修改后的 TripPlan，也不得为新实体编造 POI ID；必须区分美食级餐点与便餐，`change_meal_role` 仅接受 breakfast/lunch/dinner；
- 当前行程实体使用名称、日期、餐次和已有 ID 消歧；新实体 query 由 Coordinator 调用 AmapService 检索，先按城市和类型过滤，再按 Stage 5 唯一匹配规则解析。唯一匹配后才形成可执行 intent，无匹配或多匹配返回结构化澄清状态；对话迭代中的多匹配不影响首页额外要求“无法唯一匹配则整单 failed”的既定规则；
- “换一个酒店/餐厅/普通景点”等未指定新目标但当前对象明确的指令，允许系统排除当前实体后从真实合格候选中按既定确定性排序自动选择下一个最优候选，不要求新增候选选择 UI；
- 修改交通方式默认作用于整个行程；用户明确指定某天或具体起终点路段时，只作用于该范围。日期和路段引用必须能唯一解析，否则返回澄清状态；
- accepted 约束首版不可通过对话修改。解析结果若删除必打卡、违反饮食/时间/移动约束或以其他方式与 accepted 约束冲突，返回 `revision_constraint_conflict`；不得将其降级为普通澄清或猜测执行；
- 未知、越界或歧义指令返回结构化澄清状态，不猜测执行；
- 单次请求操作数量固定为 1；复合指令统一要求拆分。

**退出条件**：全部独立 intent、三种餐饮 intent、单动作/复合动作、当前实体消歧、新实体唯一/无/多匹配、无目标“换一个”、交通全程/日期/路段范围、歧义/越界/非法输出和 accepted 冲突均有测试；冲突或澄清指令不修改 TripPlan、不生成执行结果。

### Stage 31C：修订重算与校验闭环（独立难点 Stage）

**目标**：执行结构化修订，并保证所有受影响事实和计算结果重新生成、重新校验。

**范围**：

- 根据 intent 计算最小受影响范围，但不复用已经失效的路线、时间或预算；
- 用户新增点位导致目标日期时间不足时，允许 Coordinator 按既定修复顺序删除 `removal_rank` 最低的普通可选点为新增点让出时间；酒店、三餐、必打卡和 accepted required 目标仍不可删除。修复后必须重建全部受影响下游并通过 Validator；
- 执行前从验签通过的 TripPlan 读取完整 normalized `accepted_constraints` 并保持不变；ConstraintPolicy 先检查 intent，冲突时返回 `revision_constraint_conflict`，不进入重规划。用户若要修改 accepted 约束，只能回首页重新生成；
- 复用 TripCoordinator 的两次草案流程并按对象确定失效边界：新增或替换普通点位/美食级餐点先进入 Stage 7A 检索与 7B 筛选；只调整已有点位日期/顺序或改变 `meal_role` 从 Stage 8 重建；指定酒店先进入 Stage 9A 酒店检索与选择；指定便餐先进入 Stage 9A 对应餐次检索与选择；交通偏好变化从 Stage 10A 开始。所有路径均重算受影响路线、时间轴和预算并通过 Validator；早餐在 `lodging_side/gourmet` 间变化时从 Stage 8 重建；
- 成功后返回完整新版 TripPlan 和新完整性令牌，由前端原子替换当前状态；冲突或其他失败不产生部分更新，不追加已采纳对话消息；
- 点位、酒店、路线、时间或预算变动不得存在前端直改或跳过校验的旁路。

**不做**：不支持任意开放式工作流、多方案自动对比或未经白名单定义的修改。

**退出条件**：景点、美食级餐点、`meal_role`、酒店、便餐和交通偏好六类修订均触发正确失效范围；accepted 约束在成功修订前后逐项相同，删除必打卡及饮食/时间/移动冲突均稳定返回 `revision_constraint_conflict`。

### Stage 32：对话式迭代前端与当前会话恢复

**目标**：为 Stage 31A～31C 提供可用的持续调整入口。

**范围**：

- 报告页增加局部对话入口，不重构整个页面布局；
- 默认输入框 placeholder 提示“每次请提交一个调整，例如：把第二天的故宫换成国博”；检测到复合指令时清空本次输入并将动态 placeholder 改为“首版每次只能调整一项，请拆分后重新提交”；
- 在 Stage 13E store 上增加 `revision_no / plan_token / revision_request_id / revision_processing / draft_input / clarification / constraint_conflict / accepted_messages / revision_error`；TripPlan、revision_no 和 plan_token 必须原子替换；
- 展示用户指令、处理中状态、澄清、失败和版本更新；
- 对话 revision 出现新 POI 无匹配、多匹配或范围不明确时，不增加候选选择 UI：保留原指令在对话记录中、清空输入框，并以动态 placeholder 提示“请补充分店名、地址或所在区域后重新提交”或对应的日期/路段补充建议；用户开始输入后恢复默认 placeholder；
- `revision_constraint_conflict` 时保留输入文字但不追加为已采纳对话；在输入文字下方显示固定提示“与额外要求冲突，不予采纳”，并将整个对话输入框边框设为红色；
- 用户对输入内容发生任意修改时，立即清除该提示并恢复正常边框；这只重置前端错误态，不代表新内容已通过约束，重新提交后仍由后端权威判定；
- 新版本返回后原子替换 Pinia 当前行程，并同步时间轴与地图；
- 刷新后从 sessionStorage 恢复当前 TripPlan、修订序号和完整性令牌；对话消息和草稿只保存在当前会话；
- 关闭会话后当前结构化行程和对话可丢弃，已保存的历史 PDF 不用于恢复对话；
- 防重复提交，并处理修订序号冲突和令牌失效提示。
- 响应只有在 request ID 与当前 revision 请求一致且 base revision 仍匹配时才可应用；重复或迟到响应直接丢弃，不得覆盖较新 TripPlan。

**退出条件**：连续调整、刷新恢复、普通失败、accepted 冲突红框/文案、编辑即复位、重新提交再次冲突、关闭会话后无结构化历史的记录化人工端到端验收通过；冲突输入不进入已采纳对话，当前 TripPlan 不变；重复/迟到 request ID 不覆盖较新版本，TripPlan、revision_no 和 token 始终原子更新。

### Stage 33：统一配置管理

**目标**：最后清理已在各 Stage 随功能进入 Settings 的 LLM、高德、Unsplash、缓存、PDF 报告目录和本地前端配置。

**范围**：

- 后端统一经 `Settings` 读取、校验和脱敏展示配置；
- 前端仅保留可公开的 `VITE_*` 配置；
- 提供完整的本地开发 `.env.example` 和启动校验，不设计生产部署配置；
- 删除重复读取和不再使用的环境变量；
- 明确本地 Key 轮换与泄漏检查步骤。任何 Stage 新增配置都必须在当时进入 Settings，禁止等到本 Stage 才集中迁移。

**退出条件**：缺失必需配置时启动快速失败且提示明确；代码扫描无散落配置读取和真实密钥。

---

## 五、关键质量门禁

### 5.1 Gate A：真实数据链可用（Stage 4 后）

- POI、地理编码、天气、路线均能解析为结构化对象；
- MCP 与官方 HTTP Provider 至少一个通过能力验证并被选为默认通道；Provider 差异不会泄漏到上层 schema，人工切换 Provider 不会错误标记 degraded；
- 餐饮营业时间可解析值、缺失值和非法格式能够稳定区分，无法可靠解析时不会被猜测补全；
- 外部格式异常不会静默返回成功；
- fixture 契约测试覆盖真实响应和主要异常形态。

未通过时，不进入 Agent 新链路开发。

### 5.2 Gate B：核心规划闭环可用（Stage 13D 后）

- Agent 仅引用候选 ID；
- TripPlan 携带完整 normalized `accepted_constraints`，活动 `constraint_ids` 均可解析回约束定义；
- 明确子句被结构化为四类封闭 accepted 约束并贯穿检索、筛选、规划和校验；模糊/不支持子句只进入 ignored；任一 accepted 约束无法满足时整单 failed；
- 两次草案顺序固定为主要点位 → PrimaryDraft → 连续住宿 → 缺失餐次便餐 → 完整草案，依附关系可追溯且不存在循环依赖；
- PrimaryDraft 每日至少 1 个主要点位，明确非餐饮点 `day_part`、美食级餐点 `meal_role`、符号锚点与 provisional meal slots；美食级餐点占用正式餐次且不重复补便餐，最终午晚餐窗口和上午/下午边界一致；
- 美食偏好在 complete 中满足 1～N、在 partial 中满足 1～K 且每天最多 1 个；partial 截断前已完成跨日重排及全部下游重建；
- `lodging[0..lodging_count-1]` 连续映射返回日的起止酒店；complete 时 `lodging_count=N`，partial Day1..DayK 且 K<N 时为 K+1，A→B 后次日从 B 出发；
- 每日午餐和晚餐均恰好对应一个高德真实美食级或便餐级 POI，使用具体店名/分店名并参与活动排序、路线和时间轴；
- 路线、时间、预算均由确定性服务提供；
- 请求交通方式作为偏好先行，失效时按真实候选耗时、距离、route ID 稳定回退；accepted 移动约束始终不可放宽；
- 酒店、三餐和必打卡点不可删除；TimelineCalculator 不修改草案，Coordinator 按路线替换 → 同级餐厅替换 → 同级酒店替换 → 明确时间重排 → 删除可选点顺序修复；餐厅远近使用相邻真实路线代价，不使用直线距离；
- 预算按实际 `lodging_count`、2 名成人和 1 间房计算并输出假设；完整 N 天为 N 晚，partial Day1..DayK 为 K+1 晚；
- 1～3 日计划满足早餐、用餐、缓冲、非末日抵达结束酒店和末日晚餐结束边界；删点后 PrimaryDraft 及全部下游已重建；
- `complete / partial / failed` 与 `degraded` 语义正确；
- partial 仅包含从首日开始的最长连续完整前缀，住宿链和预算随前缀重算；
- 首末日判定基于请求日期而非返回数组位置：complete 的请求末日适用 17:00—18:00 晚餐和无返店规则，partial K<N 的所有返回日均为非末日；
- 美食级/便餐级午晚餐有来源营业时间时，候选餐窗内存在连续 ≥40min 可用区间；美食级早餐不做固定餐窗预筛。所有最终实际餐次均被有来源营业时间覆盖，缺失时显式标记 unavailable 并提示核实；
- 新链路 shadow 结果通过，当前 Vue 前端的新 schema 适配就绪；旧链路短期回滚开关仍可工作，此 Gate 不单独触发默认切换。

未通过时，不进入大规模前端功能开发。

### 5.3 Gate C：开发环境可靠链路可用（Stage 20 后）

- 无虚构 fallback；
- 缓存与 stale 降级可追溯；
- 错误分级、天气范围和地图路线真实性通过测试；
- 有效天气先影响 Stage 7B 室内外点位筛选/替换，再由 Stage 8 分天排序；天气导致的点位变化已重建全部下游；
- 前后端 Key 隔离，控制台限制已完成。
- 新 TripCoordinator 链路已在 Stage 16 切为默认，默认开发路径不存在虚构 fallback 和手动编辑旁路。

### 5.4 Gate D：P2 体验闭环（Stage 30 后）

- 用户无须预先展开天级卡片即可导出；程序自动全展开生成一个 PDF Blob，完成浏览器下载后将同一 Blob 保存到固定目录，重新打开网页时可列出并下载；未导出行程不进入历史；SSE 反映真实状态；
- 图片、导出、代理、折叠和 PC 显示回归通过；
- 1～3 天天级卡片可全部展开，PDF 中图片、地图和跨页内容完整；所有外部查询保持同步串行。

只有 Gate D 通过后才启动 P0-9。

### 5.5 Gate E：对话迭代可上线（Stage 32 后）

- 所有实质调整走后端重算和校验；
- 景点/美食级/meal_role、酒店、便餐和交通偏好修订分别从 Stage 8、9A 住宿、9A 餐饮或 10A 的正确边界失效并重建；
- accepted 约束由 plan_token 绑定且首版不可修改；冲突输入返回稳定错误，不改变 TripPlan、不进入已采纳对话，并呈现指定红框和提示文案；
- 单次只执行一个 revision 动作；复合指令和实体/范围歧义只进入澄清展示，不产生部分修改。澄清通过动态 placeholder 指导用户重新输入，不增加候选选择 UI；
- 修订失败不覆盖当前行程，刷新时当前会话恢复可靠；
- 前端没有仅修改本地正式数据的旁路。

---

## 六、测试策略

### 6.1 后端轻量测试

- 只引入 `pytest`，不配置覆盖率门槛、CI、测试报告平台或专用复杂 mock 框架；
- 使用普通 Python fake 替代 LLM、MCP 和 HTTP；解析契约使用脱敏 MCP/官方 HTTP fixture；
- 必测自动化范围：schema、解析器、饮食规则、去重、时间轴、预算、Validator、路线回退、修复终止和 partial/failed 决策；
- API 错误、令牌完整性及 SSE 正常/失败事件可使用 FastAPI 自带测试能力；PDF、地图和真实断连不强制自动化；
- 在线冒烟只在开发者显式配置 Key 时手动运行，不作为普通测试命令的稳定依赖。

### 6.2 前端检查与人工验收

- 类型检查与生产构建；
- 不强制引入 Vitest、Vue Test Utils 或 Playwright；
- Pinia、时间轴、状态提示、折叠、图片失败、地图、PDF 和聊天状态采用记录化人工验收清单；
- 核心人工流程：生成 → 首页/行程页双向跳转 → 导出 PDF（本地下载 + 固定目录保存）→ 重新打开网页 → 历史下载 → 当前行程对话调整。

### 6.3 必测业务边界

1. 1 日游计 1 晚酒店，同时适用末日晚餐 18:00 前结束且无返店段规则；
2. 非末日住宿侧早餐 09:00～21:00 并抵达 `end_hotel_id`；
3. 非末日美食级早餐 08:00～21:00；
4. 末日两种早餐均在 18:00 前完成晚餐并结束旅程，不生成返店路线；
5. 每段路线外存在至少 30 分钟独立缓冲；
6. 每顿在外用餐至少 40 分钟；
7. 多偏好全部检索；美食偏好开启时 complete 为 1～N 个、partial 为 1～K 个且每天最多 1 个美食级餐点；缺少时先跨日重排，仍无法满足则 failed；关闭时不存在美食级餐点；
8. 超过 3 天、超过 3 个偏好、日期天数不一致和非法枚举均被拒绝；
9. POI 无结果、路线缺失、天气超范围、LLM 返回未知 ID/额外字段；
10. 交通偏好候选不可用时按定义回退集合选择 `(duration, distance, route_id)` 最小的真实路线；walking 仅在真实路线 ≤2km 时可选，3km 路段自动选择真实 taxi/transit，不会仅因步行超限 failed；transit 实际步行接驳段也满足适用上限；
11. 默认 2 名成人、1 间房、实际 `lodging_count` 晚；完整计划 N 晚，partial 为 K+1 晚；门票/餐饮/公交按人、打车按车，页面假设与预算明细一致；
12. 酒店、三餐和必打卡点不可删除；普通天级失败只返回首日起最长连续前缀，首日失败或任一 accepted 约束无法满足时整单 failed；
13. 从全部折叠、部分展开和全部展开状态导出的 PDF 内容一致且包含所有天，导出后恢复原展开状态；
14. `complete + degraded`、`partial + degraded`、`failed` 的前后端一致展示；
15. 同一当前行程的重复/迟到修订响应不能静默覆盖较新结果；
16. 除住宿侧早餐外，午餐和晚餐必须各自恰好引用一个高德真实美食级或便餐级 POI，显示具体店名/分店名；美食级餐点按 `meal_role` 替代同餐次便餐，“川菜馆”“附近餐厅”等泛化名称、重复餐次和未进入路线/时间轴的餐点均被拒绝；
17. “龙门石窟是必打卡点”解析为 trip-scope `must_visit_poi`，可安排在任意一天；无法唯一匹配或最终未安排时整单 failed；
18. “我不吃辣”解析为 `dietary_exclusion=avoid_spicy`，固定排除川菜、湘菜、重庆火锅、麻辣烫等规则表类别，即使餐馆可能有不辣菜品也不恢复；
19. 明确时间约束字段不全、无法验证的完整无障碍要求及其他模糊子句进入 ignored，不触发检索、不进入后续 Prompt；accepted 明确时间约束同时产生 required 目标并必须进入行程；可按路线距离验证的移动限制正常执行；
20. A→B→B 住宿链满足：第 1 天 A 出发 B 结束，第 2 天 B 出发 B 结束，第 3 天 B 出发且末日无结束酒店；所有换店末段均有真实路线；
21. TimelineCalculator 对输入无副作用；Coordinator 按替换路线、替换同级餐厅、明确时间重排、删除可选主要点的顺序修复并重建下游；
22. 午餐完整落在 12:00—14:00，普通日晚餐完整落在 18:00—20:00，末日晚餐完整落在 17:00—18:00；day_part 与最终时刻一致，非餐饮主要点不跨午餐，美食级午餐作为午餐活动本身通过校验；
23. 只有全日期跨日重排及有限修复仍失败后才允许生成 partial；日期类型按 `date == request.end_date` 判定。Day2 失败时三日请求只返回 Day1，Day3 失败时返回 Day1～2；这些返回日全部按非末日规则并抵达对应结束酒店，Day1 失败时整单 failed。
24. 每个完整日至少 1 个景点或美食级主要点，便餐不计；零主要点位日期不可作为完整日返回。
25. `avoid_spicy` 等饮食禁忌过滤全部系统选择的真实餐饮 POI，但不校验由用户自行选择的住宿侧早餐。
26. 合法空候选按业务不可行进入 partial/failed；技术超时且重试、缓存均不可用时返回 API 错误；stale 成功时返回计划并标记 degraded。
27. 美食级/便餐级午晚餐有来源营业时间时，候选阶段须在固定餐窗内存在连续 ≥40min 可用区间；美食级早餐跳过候选预筛，并在时间轴生成后校验营业时间覆盖实际早餐区间。缺失时保留 null/unavailable 并提示核实。
28. 额外要求解析失败且重试耗尽时进入 `parse_failed` ignored 并继续；PrimaryDraft、ItineraryDraft、RouteSelection 等核心输出解析失败且重试耗尽时返回稳定 API 错误。
29. 恶劣天气可从真实候选池替换室外主要点，普通天气可只调整日期/顺序；两种变化均从 PrimaryDraft 重建全部下游，超预报范围不产生伪天气决策。
30. 对话修订中，美食级餐点或 `meal_role` 变化从 Stage 8、酒店变化从 Stage 9A 住宿、便餐变化从 Stage 9A 餐饮、交通偏好变化从 Stage 10A 重建。
31. Stage 8 初次交付时天气字段对决策零影响；Stage 18 上线后天气先进入 Stage 7B 再进入 Stage 8，SSE 顺序与真实调用一致。
32. 无下午点且午餐待选时使用 `LUNCH_SLOT`；选定午餐后解析为真实 POI ID，再检索晚餐，非法使用 `DAY_END` 的草案被拒绝。
33. 额外要求解析、候选筛选和 revision intent 使用 temperature=0；PrimaryDraft、ItineraryDraft、RouteSelection 使用 temperature=0.3。
34. 首版三种餐饮 revision intent 均可执行；accepted 约束篡改、删除必打卡和饮食/时间/移动冲突均返回 `revision_constraint_conflict`，不改变行程。
35. 前端收到约束冲突后保留输入，显示“与额外要求冲突，不予采纳”及红色输入框边框；任意编辑立即清除错误态，重新提交仍以后端结果为准。
36. 酒店导致必要路线缺失或时间轴不可行时替换同住宿类型酒店，并使相邻日期边界、餐次、路线、时间轴和预算正确失效重建。
37. 餐厅替换按相邻真实路线总耗时、总距离和 POI ID 排序，不使用直线距离；公交实际出发时间偏差超过 30 分钟时最多重查一次。
38. TimelineCalculator 的 `wait` 与强制 `buffer` 独立，问题顺序稳定；相同输入产生相同时间轴且不修改输入。
39. 费用使用 CNY Decimal；缺失项不进入 `known_cost_total`，前端明确展示“已知费用合计”和缺失费用。
40. SSE 客户端断开后，当前外部调用结束即停止，无后续阶段、无后台任务、无保存 TripPlan。
41. `lodging_side` 早餐计入三餐齐全但没有 POI、路线或外出事件；gourmet 早餐引用真实 POI 并完整进入路线、时间轴、营业时间和预算校验，二者不得并存。
42. 景点和餐饮建议时长缺失或越界触发 Agent 输出重试；Calculator 不猜默认值，修复不压缩到类型下限以下。
43. POI、地理编码、polyline、缓存和地图统一使用 GCJ-02；经纬度颠倒、非法范围和坐标系不一致被拒绝。
44. revision 新实体只输出 query，经 AmapService 唯一匹配后执行；无匹配或多匹配返回澄清，不生成 POI ID。新增/替换与仅调整现有实体使用不同失效边界。
45. PDF 并发上传由服务端生成不同安全文件名，临时文件原子提交；失败和扫描过程不暴露半文件。
46. revision 复合指令不拆分、不部分执行，TripPlan 保持不变；输入框动态 placeholder 提示用户每次只提交一个调整。
47. 新增点位导致时间不足时可按 `removal_rank` 删除普通可选点，但不得删除酒店、三餐、必打卡或 accepted required 目标，并须重建下游。
48. “换一个”且当前对象明确时排除当前实体并自动选择下一个真实合格候选；没有合格候选时修订失败且当前行程不变。
49. 未指定范围的交通修改作用于整个行程；明确指定某天或具体路段时只修改该范围，无法唯一解析范围时进入澄清。
50. 对话 revision 的新 POI 多匹配不展示候选选择 UI、不执行修改；保留原指令记录、清空输入框，并用 placeholder 提示补充分店名、地址或区域。首页额外要求的必打卡多匹配仍整单 failed。

---

## 七、发布与回滚策略

1. **按 Stage 合并**：每个 Stage 单独提交/合并，不将多个高风险 Stage 打成一次大发布。
2. **短期回滚切换**：Stage 5～15 期间新链路只走独立测试入口或显式功能开关；Stage 16 将新前后端同步切换为默认。没有外部调用方，不建设长期旧 schema 兼容层，旧实现只保留短期回滚开关。
3. **能力开关**：SSE、历史和对话迭代分别使用独立功能开关，便于小流量验证和快速关闭。
4. **报告文件兼容**：PDF 元数据格式只做向前兼容调整；回滚应用时不得删除或覆盖已有报告文件，程序不自动清理，目录由开发者手动管理。
5. **外部服务保护**：外部查询保持同步串行，并配置超时、限流和调用统计；发现配额异常时收紧调用预算，不放宽真实性和失败规则。
6. **旧代码清理时机**：旧 `MultiAgentTripPlanner`、旧 schema、假进度和 sessionStorage 唯一路径，分别在对应替代能力稳定后单独清理，不夹带在功能 Stage 中。

---

## 八、优化项覆盖检查

| 优先级 | 覆盖情况 |
|--------|----------|
| P0 | P0-1：1、3～4；P0-2：5；P0-3：5～6；P0-4：2、5、7B、8、9B；P0-5：7A～7B、9A～9B；P0-6：10A～10B；P0-7：7A～13D（含 9A/9B、12A/12B）；P0-8：11、14；P0-9：31A～32 |
| P1 | P1-1：16；P1-2：15；P1-3：17；P1-4：20；P1-5：18；P1-6：19 |
| P2 | P2-1：13E、26～27；P2-2：28；P2-3：25；P2-4：23；P2-5：24；P2-6：21；P2-7：22；P2-8：29；P2-9：30 |
| P3 | P3-1：33 |

Future 项仅保留接口扩展空间，本计划不承诺实现：RouteService 不加入高级多目标策略，餐饮不接第三方榜单，不实现多方案对比和行程分享。

---

## 九、后续技术验证项

以下事项不改变本轮已确认的产品语义，但应在对应 Stage 开始前通过实测定稿：

1. Stage 1 实测后，在 MCP 与高德官方 HTTP Provider 中选定当前默认通道，并确认串行稳定性、路线能力和调用预算；
2. Stage 5 前，验证目标模型对 JSON mode/`response_format` 的实际兼容性；不改变“字符串适配 + `extra="forbid"` 校验 + 有限重试”的既定方案；
3. Stage 17 前只需根据 Stage 1 实测确定进程内缓存的 TTL、容量和淘汰策略，缓存介质已确定；
4. Stage 26 前确认可持久化固定磁盘的实际路径和上传大小上限；程序不自动清理，开发者手动管理目录，本计划不引入 SQLite；
5. Stage 28 使用 `POST + fetch ReadableStream`；实现前只需验证本地浏览器兼容性。断连后当前外部调用结束即停止，不保存结果、不启动后台任务；
6. Stage 31B 前，评审首版 revision intent 白名单，避免以“自然语言无限能力”扩大范围。

上述决策均应形成简短 ADR，并回填本计划；不得把尚未验证的假设直接固化为默认实现。
