# Stage 3：AmapService POI 与地理编码结构化解析
> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 3 |
| 目标 | 将 Stage 1 选定 Provider 的 POI/地理编码原始结果，确定性解析为 Stage 2 的真实来源模型，彻底区分合法空结果、可选字段缺失和上游非法响应 |
| 对应优化项 | P0-1 AmapService 结构化解析 |
| 前置依赖 | Stage 1、Stage 2 |
| 后续阻断 | 未通过时不得开始 Stage 4、5、6、7A、9A、13A、17 |
| 默认流量影响 | 无；只接入新 AmapService 结构化入口及离线契约测试，旧 Agent 和旧 `/trip/plan` 仍使用原链路 |

## 2. 当前代码基线与差异

- 当前入口是 `backend/app/services/amap_service.py`。`AmapService.__init__()` 直接取得全局 MCPTool，没有注入 Stage 1 Provider 的边界，也无法用 fixture/fake 隔离测试。
- `search_poi()` 调用 `maps_text_search` 后只打印原始结果前 200 字符，固定返回 `[]`；合法空结果、Provider 异常、响应格式漂移和真实“有结果”在调用方看来完全相同。
- `geocode()` 调用 `maps_geo` 后固定返回 `None`；坐标顺序、范围、坐标系、匹配数量和城市约束均未校验。
- `get_poi_detail()` 使用正则从任意文本中贪婪提取 `{...}`，成功时返回无 schema 的字典、失败时返回 `{"raw": result}` 或 `{}`；该路径会泄漏 Provider payload 形状，且不能稳定识别嵌套/多段 JSON。
- 当前 `POIInfo` 只包含 ID、名称、类型、地址、坐标和电话；缺少 provider、采集时间、freshness、quality、原始类型、行政区、评分、票价和营业时间等来源语义。
- 当前异常被捕获后打印并转换为空容器，日志可能包含未经脱敏的上游文本；目标行为必须传播 Stage 1 的稳定 ProviderError，不把技术故障伪装成业务空结果。
- 当前没有 POI/地理编码解析器、营业时间解析器、逐条诊断、脱敏 fixture 契约测试或 FakeAmapProvider 注入入口。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 修改 | `backend/app/services/amap_service.py` | 改为构造函数注入 `AmapProvider`；新增结构化 `search_poi()`、`geocode()` 入口；天气/路线保留给 Stage 4，旧详情入口保持 legacy 隔离 |
| 新增 | `backend/app/services/amap_parsers/__init__.py` | 仅导出 POI、地理编码和营业时间稳定解析入口 |
| 新增 | `backend/app/services/amap_parsers/adapters/__init__.py` | 根据 Provider 名称选择唯一 payload 提取适配器 |
| 新增 | `backend/app/services/amap_parsers/adapters/mcp.py` | 从 MCP payload 提取 POI/地理编码规范化中间记录，不生成领域事实 |
| 新增 | `backend/app/services/amap_parsers/adapters/http.py` | 从 HTTP payload 提取 POI/地理编码规范化中间记录，不生成领域事实 |
| 新增 | `backend/app/services/amap_parsers/poi.py` | 将 ProviderRawResult 解析为 `SourcedPOI`，处理必需/可选字段、类型、行政区与稳定顺序去重 |
| 新增 | `backend/app/services/amap_parsers/geocode.py` | 解析地理编码候选、地址层级、城市/adcode 与 GCJ-02 坐标，不替调用方决定唯一匹配 |
| 新增 | `backend/app/services/amap_parsers/opening_hours.py` | 只解析 Stage 1 能力矩阵确认可确定识别的营业时间形态，其余返回 unavailable 诊断 |
| 新增 | `backend/app/services/amap_parsers/errors.py` | 定义 `AmapParseError` 与逐条诊断；与 ProviderError 分层，复用稳定 error_code，不重复定义 API 错误响应 |
| 修改 | `backend/app/models/source.py` | 仅在 Stage 1 实测字段需要时补齐 POI/地理编码来源字段；不得改变 Stage 2 已冻结的真实性、null 和坐标语义 |
| 新增 | `backend/tests/contract/test_amap_poi_parser.py` | 使用 Stage 1 MCP/HTTP 脱敏 fixture 验证两通道输出同一领域语义 |
| 新增 | `backend/tests/contract/test_amap_geocode_parser.py` | 验证单条/多条/空结果、城市约束、坐标与格式漂移 |
| 新增 | `backend/tests/unit/services/test_opening_hours_parser.py` | 验证缺失、单段、分段、跨午夜、`24:00`、星期和非法格式 |
| 新增 | `backend/tests/unit/services/test_amap_service.py` | 用 FakeAmapProvider 验证调用参数、返回、错误传播、顺序去重与日志脱敏 |

旧 `backend/app/models/schemas.py`、`backend/app/agents/trip_planner_agent.py` 和 API 路由本 Stage 不修改。新 AmapService 不得反向返回旧 `POIInfo`，旧链路继续使用其原入口，直到 Stage 5 shadow 接入和 Stage 16 默认切换。

## 4. 输入、输出与错误契约

### 4.1 服务方法

| 方法 | 输入 | 成功输出 | 合法空结果 |
|------|------|----------|------------|
| `search_poi` | `request_id`、`keywords`、`city`、`city_limit=true`、可选类型过滤、`page_size=20`；字符串非空，page_size 只允许 1～20 | `list[SourcedPOI]`，保持来源顺序并按 `(provider, source_id)` 稳定去重 | `[]` |
| `geocode` | `request_id`、`address`、可选 `city`；地址非空 | `list[GeocodeCandidate]`，保留全部合法匹配及行政区信息 | `[]` |

- AmapService 只依赖统一 `AmapProvider`，不得分支判断 MCP/HTTP payload；`ProviderRawResult.provider` 只用于 registry 选择 MCP/HTTP payload 提取适配器，之后统一进入 provider-agnostic POI/geocode parser。
- Provider 调用保持同步串行；本 Stage 不增加并发、缓存、重试或运行期 Provider 切换。
- `geocode()` 返回候选列表而不是擅自取第一条。唯一匹配决策由 Stage 5 的必打卡解析或后续调用方根据 city/adcode 明确完成。
- 当前 legacy `get_poi_detail()` 不属于 Stage 1 最小 Provider 契约，新链路 Stage 3 不调用也不重写它；其原始字典行为不得进入 Coordinator，后续若新链路确需详情能力，必须单独扩展 Provider 契约并评审。

### 4.2 POI 必需字段与可选字段

| 字段 | 规则 |
|------|------|
| `source_id` | 必须来自高德响应且非空；禁止用名称、数组下标或哈希补造 |
| `name` | 必须为具体返回名称并非空；餐饮保留分店名，禁止使用搜索词或“餐厅/川菜馆”等类别代替实体名 |
| `address` | 必须来自响应且非空；若目标 Provider 对某类 POI 确认可能缺失地址，该实体不得进入正式候选，只能产生逐条 invalid-item 诊断 |
| `location` | 必须是合法 GCJ-02 `longitude/latitude`；禁止经纬颠倒、默认城市坐标、算法平移或 LLM 修复 |
| `poi_type/raw_type` | 保留来源原始类型，并映射到 Stage 2 受控领域类型；未知类型不得猜测，可标为受控 `unknown`，但不得丢弃原始类型 |
| `provider/fetched_at/freshness/query_scope` | 从 ProviderRawResult、调用参数和当前注入时钟确定；实时结果为 sourced + live，query_scope 记录请求 city/city_limit 及来源可用 adcode，不由 Agent 填写 |
| `adcode/district/tel/rating/ticket_price/opening_hours` | 均为可选来源事实；缺失为 null/unavailable，不使用空字符串或 0 补齐 |

- 单个 POI 的必需字段缺失或非法时，不输出半真实体；记录脱敏 `invalid_item` 诊断并跳过该项。
- 响应 envelope 合法且明确无 POI 时返回 `[]`。响应声称有数据但所有条目都因必需字段非法被跳过时，抛出 `invalid_response`，不得降格为空结果。
- 部分条目合法、部分非法时返回合法条目并附服务内部诊断；诊断进入结构化日志/测试证据，不进入 Agent 候选事实，也不携带完整原始条目。
- 同一响应重复 `source_id` 时保留第一条并记录 duplicate 诊断；同名但不同 source ID（包括不同分店）保留为不同实体。

### 4.3 地理编码与坐标

- `GeocodeCandidate` 至少包含规范化地址、GCJ-02 坐标、provider/source metadata，以及来源提供时的 province/city/district/adcode/level。
- Provider 原始坐标系必须显式记录。若 Stage 1 证明原始结果不是 GCJ-02，只能由 Provider 层执行已验证转换并记录转换元数据；AmapService 不猜测坐标系。
- 坐标字符串只接受 Stage 1 验证过的明确格式；必须先拆为 longitude、latitude 再分别做范围校验。顺序颠倒、数量不等于 2、非数字或越界均为 invalid item。
- city/adcode 是匹配约束而不是描述文本。返回候选与显式 city 明确冲突时剔除并记录诊断；无法从条目字段证明一致时，只有 Stage 1 ADR 已证明目标 Provider 严格执行 city_limit，且 query_scope 记录 `city_limit=true` 时，才允许以查询作用域证明候选归属，否则保留为无法证明并供上层拒绝。
- 多候选保持 Provider 顺序并按来源 ID 或“规范化地址 + 坐标”稳定去重；不得由 AmapService 静默选择评分最高或第一条作为唯一结果。

### 4.4 营业时间

- 解析器只接受 Stage 1 fixture 中已验证的字段位置和格式，输出 Stage 2 定义的星期 + 多时间段结构。
- 明确缺失、空值或 Provider 不提供时返回 `value=null / quality=unavailable / reason=missing`；格式存在但不在可确定解析集合时使用 reason=`unparseable`。
- 支持的语义必须区分：单段、同日分段、星期范围、跨午夜和 `24:00`；不得把跨午夜截断成同日，也不得用默认营业时段代替缺失值。
- Stage 3 只负责语法规范化，不判断某餐厅是否覆盖午晚餐窗口或实际用餐区间；候选窗和最终时间轴校验由 Stage 7A/9A 与 TimelineValidator 承担。

### 4.5 错误与日志

- Provider 的 configuration/authentication/rate_limited/timeout/upstream_error/invalid_response 继续以 Stage 1 `ProviderError` 原样分类传播，AmapService 不转换为空集合。
- 解析层使用 `AmapParseError(error_code="invalid_response")`：用于 envelope 漂移、类型不符、声称有数据但无任何合法必需实体等情况。它与 ProviderError 类型不同但稳定 error_code 相同，Stage 15 统一映射 502；合法空结果不是错误。
- 错误上下文最多包含 request_id、provider、operation、稳定 reason 和条目序号/字段名；禁止记录 Key、完整 URL、完整 payload、用户完整自由文本或未经脱敏的 POI 原始条目。
- 本 Stage 不决定 HTTP 状态码；Stage 15 按统一错误契约将 invalid_response 映射 502、配置/上游/超时映射相应 API 状态。Stage 5 shadow 链路必须能够区分这些错误与业务空结果。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或用户流量开关；沿用 Stage 1 的 `AMAP_PROVIDER`、`AMAP_API_KEY` 和 Provider factory。
- AmapService 构造时必须显式接收 Provider；应用装配层可调用 factory，单元测试只注入 FakeAmapProvider，业务方法不得自行读取 Settings 或环境变量。
- 页大小上限、Provider 支持的类型过滤参数和详情能力以 Stage 1 ADR 为准，作为受控代码常量或 Provider capability，不在环境变量中制造可漂移业务语义。
- `fetched_at` 使用可注入 UTC 时钟并序列化为明确时区时间；测试不得通过 sleep 或系统当前时间做脆弱断言。
- 不新增缓存 TTL、重试次数或请求并发配置；分别留给 Stage 17、Stage 15/16 和本阶段明确不做范围。

## 6. 实施步骤

1. 依据 Stage 1 ADR 固定 MCP/HTTP POI、地理编码 payload 的适配入口；先为每种已选通道建立最小成功、空结果和错误 fixture 映射。
2. 实现共享解析诊断、字符串/数值/null 规范化和 GCJ-02 坐标解析；所有解析函数保持纯函数并接收 ProviderRawResult 与注入时钟。
3. 实现 POI envelope 与逐条解析：校验必需字段、保留可选来源字段、映射受控类型、稳定去重，并区分空结果、部分坏条目和整体 invalid_response。
4. 实现地理编码候选解析：保留全部合法候选、行政区和 adcode，校验显式城市冲突，不替上层选择唯一结果。
5. 实现 Stage 1 已验证范围内的营业时间解析；缺失与不可解析都输出 unavailable，但使用不同稳定 reason。
6. 改造新 AmapService 为 Provider 注入式结构化入口，传播 ProviderError/AmapParseError；legacy 详情方法保持隔离且禁止新链路调用。
7. 增加 FakeAmapProvider 服务测试与双通道 fixture 契约测试，验证日志/诊断脱敏、新旧模型隔离和天气/路线仍由 Stage 4 接管。
8. 执行 Stage 1、Stage 2 与 Stage 3 定向测试、全量后端回归和前端构建，归档字段矩阵与回滚证据。

## 7. 明确不做

- 不实现天气、路线或 route polyline 解析，不修改 `get_weather()`、`plan_route()` 的新链路职责；这些属于 Stage 4。
- 不实现偏好到关键词映射、多偏好循环、候选上限、Agent 筛选、必打卡唯一匹配、饮食过滤或酒店/便餐锚点策略。
- 不判断营业时间是否覆盖固定餐窗或实际用餐区间，只做来源语法规范化。
- 不增加缓存、stale 降级、自动重试、并发请求、Provider 自动切换或性能优化。
- 不改旧 Agent Prompt、旧 `schemas.py`、API 路由、前端类型和页面，不把新结构化结果接入默认 `/trip/plan`。
- 不用正则猜取任意 JSON、不保留 `{"raw": ...}` 兜底、不用 LLM/默认坐标/名称哈希修复事实。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S3-T01 | contract | MCP 与 HTTP 成功 POI fixture | 输出同一 `SourcedPOI` 领域语义；通道特有字段不泄漏到上层 |
| S3-T02 | contract | 合法空 POI fixture | 返回 `[]`，不抛异常、不生成占位实体 |
| S3-T03 | contract | POI 必需字段完整、可选字段缺失 | 必需事实保留；可选事实为 null/unavailable 且 reason 稳定 |
| S3-T04 | contract | 部分条目缺 ID/名称/地址/坐标 | 只返回合法条目，坏条目产生脱敏 invalid_item 诊断 |
| S3-T05 | contract | 声称有数据但所有条目非法、envelope 漂移 | 抛出 `invalid_response`，不降格为空列表 |
| S3-T06 | contract | 重复 ID、同名不同 ID 及餐饮分店 | 同 ID 保留第一条并记录 duplicate；不同 ID/分店全部保留 |
| S3-T07 | contract | 单条、多条与空地理编码 fixture | 保持来源顺序；多条不自动选第一条；空结果返回 `[]` |
| S3-T08 | contract | 坐标颠倒、非数字、越界、数量错误与坐标系异常 | 非法条目拒绝；不生成默认坐标或算法修复坐标 |
| S3-T09 | contract | 显式 city/adcode 一致、冲突和无法证明一致 | 一致通过，明确冲突剔除，未知保留来源字段供上层判断 |
| S3-T10 | unit | 营业时间缺失、单段、分段、星期范围 | 合法格式规范化；缺失为 unavailable/missing |
| S3-T11 | unit | 跨午夜、`24:00` 与未知格式 | 合法边界不截断；未知格式为 unavailable/unparseable，不猜默认时间 |
| S3-T12 | unit | FakeAmapProvider 成功、空结果和六类 ProviderError | 调用参数正确；错误分类原样传播；技术错误不返回空集合 |
| S3-T13 | unit | request_id 与 query_scope | request_id 原样传播但不进入查询条件；city/city_limit/adcode 来源作用域可审计 |
| S3-T14 | unit | 注入时钟、诊断与日志 | fetched_at 可重复；日志只含稳定上下文，不含 Key、完整 payload/URL |
| S3-T15 | contract | 新旧模型和 Stage 4 边界 | 新入口只返回 Stage 2 来源模型；旧模型仍可导入；天气/路线未被本 Stage 假实现 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/contract/test_amap_poi_parser.py backend/tests/contract/test_amap_geocode_parser.py backend/tests/unit/services/test_opening_hours_parser.py backend/tests/unit/services/test_amap_service.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实高德服务。Stage 3 必测用例数为 15，失败数必须为 0；实时 Provider 验证只引用 Stage 1 已归档证据，不在默认测试中重复消耗配额。

## 9. 人工验收

不适用：Stage 3 不接入 API/UI，解析、错误和日志行为均使用 Stage 1 脱敏 fixture 与 FakeAmapProvider 自动化覆盖。评审者在 `field-matrix.md` 中核对 Stage 1 实测字段到 Stage 2 模型的逐项映射，不重新运行真实 Key 请求。

## 10. 量化退出指标

- [ ] S3-T01～S3-T15 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 两种 Provider 对同类事实输出相同领域语义；默认 Provider 的每个输出 POI 均有真实 source ID、非空具体名称/地址和合法 GCJ-02 坐标。
- [ ] 合法空结果、部分坏条目和整体非法响应分别稳定映射为 `[]`、合法子集 + 诊断、`invalid_response`，三者互不冒充。
- [ ] 地理编码不静默取第一条；单条、多条、城市冲突、坐标顺序/范围和空结果均有确定行为。
- [ ] 营业时间的 missing、unparseable、单段、分段、星期、跨午夜和 `24:00` 均有 fixture；任何未知格式都不由 LLM 或默认值补写。
- [ ] 同一来源 ID 稳定去重，同名不同 ID 和分店不合并；检索词/类别不能成为餐饮实体名称。
- [ ] ProviderError 与解析错误不被转换为空容器；日志、诊断、异常和证据中的真实 Key、完整 payload、完整 URL 命中数为 0。
- [ ] AmapService 可注入 FakeAmapProvider 和时钟；默认测试公网请求数为 0，新增未追踪 TODO 数为 0。
- [ ] 旧 `/trip/plan`、旧 schema 和前端构建不受影响；天气/路线解析职责明确留在 Stage 4。
- [ ] 03 Stage 3 的每条退出条件和 02 的 POI/坐标/营业时间真实性规则均映射到测试或证据 ID。

## 11. 迁移与回滚

- Stage 3 的新结构化入口不接默认流量；回滚恢复 `amap_service.py` 的旧入口引用并删除新增解析器/测试即可，不涉及 TripPlan、前端状态、缓存或持久化数据迁移。
- `models/source.py` 只允许增量兼容字段；若后续 Stage 已依赖新增字段，必须按依赖逆序回滚，禁止单独删除字段制造半兼容模型。
- 回滚后旧 Agent 与 `/trip/plan` 必须保持原行为；Stage 1/2 测试、旧模型导入检查和前端生产构建必须通过。
- 失败 fixture、字段缺口和 Provider capability 结论属于决策证据，不随代码回滚删除，避免后续再次以猜测实现同一字段。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_3/test-summary.md` | 提交标识、三条命令、15 个用例结果、离线声明和失败数 |
| 字段矩阵 | `doc/opt_strategy/evidence/stage_3/field-matrix.md` | Provider/operation/fixture 字段 → parser → Stage 2 模型字段 → quality/reason → 测试 ID |
| 错误矩阵 | `doc/opt_strategy/evidence/stage_3/error-matrix.md` | 空结果、坏条目、AmapParseError invalid_response 与 ProviderError 的稳定行为 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_3/rollback-check.md` | 回滚范围、依赖逆序要求、验证命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计复核结论：**R2～R10 通过首轮跨 Stage 审计；执行 Ready 待 R1**。R1 必须等待 Stage 1、2 实际 Done，并在 Stage 3 开工前再次确认 ProviderRawResult、来源模型与解析适配器契约未变化；当前不表示 Ready、Done、设计冻结或允许修改工程代码。
