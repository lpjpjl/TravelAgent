# Stage 4：AmapService 天气与路线结构化解析

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 4 |
| 目标 | 将 Stage 1 选定 Provider 的天气与步行/驾车/公交路线原始结果，确定性解析为 Stage 2 来源模型，补齐 AmapService 的外部事实入口 |
| 对应优化项 | P0-1 AmapService 结构化解析，并为 P0-6、P1-4、P1-5 提供事实基础 |
| 前置依赖 | Stage 1、Stage 2、Stage 3 |
| 后续阻断 | 未通过时不得开始 Stage 5、10A、10B、11、13A、18、20、28 |
| 默认流量影响 | 无；只扩展新 AmapService 结构化入口及离线契约测试，旧 Agent 和旧 `/trip/plan` 仍使用原链路 |

## 2. 当前代码基线与差异

- 当前 `backend/app/services/amap_service.py` 的 `get_weather()` 调用 MCP 后只打印结果前 200 字符并固定返回 `[]`；真实预报、合法空结果、ProviderError 和格式漂移无法区分。
- 当前 `WeatherInfo` 使用任意字符串日期和天气字段；温度允许 int/string，解析失败回退为 `0`，会把 unavailable 伪装成真实零度；没有来源、采集时间、预报范围或 freshness。
- 当前 `plan_route()` 接收地址和可选城市，按字符串选择 walking/driving/transit MCP 工具；未知 route_type 静默回退 walking，无法拒绝无效输入。
- 当前路线调用成功后固定返回 `{}`，失败也捕获为 `{}`；距离、耗时、polyline、route ID、策略和换乘信息均未解析。
- 当前地址路线入口与 Stage 1/2 的 GCJ-02 坐标事实边界不一致。目标新入口以合法坐标为主键；地址到坐标必须先经 Stage 3 地理编码，不允许路线解析器再次模糊解析地址。
- 当前 `RouteInfo` 只有 distance、duration、route_type 和自由文本 description，不能表达来源、起终点、策略、polyline、原生/生成 ID、公交段或驾车计费输入。
- 当前没有天气、路线解析器，没有三类路线脱敏 fixture 契约测试，也没有“请求日期超出预报范围”的 unavailable 语义。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 修改 | `backend/app/services/amap_service.py` | 在 Stage 3 注入式服务上新增 `get_weather()` 与 `plan_routes()`；删除新入口的空列表/空字典错误吞并 |
| 新增 | `backend/app/services/amap_parsers/weather.py` | 解析预报 envelope、日期、昼夜天气/温度/风力和来源，按请求日期补 unavailable |
| 新增 | `backend/app/services/amap_parsers/route.py` | 解析步行、驾车、公交候选的距离、耗时、策略、polyline、原生 ID 和方式事实 |
| 新增 | `backend/app/services/amap_parsers/polyline.py` | 解析并校验 GCJ-02 路径点，拼接步骤路径、去除相邻重复点但不简化/猜补路线 |
| 新增 | `backend/app/services/amap_parsers/route_id.py` | 在无原生 ID 时按冻结的规范化输入与路线摘要生成请求级确定性 ID |
| 修改 | `backend/app/services/amap_parsers/adapters/mcp.py` | 增加 MCP 天气与三类路线 payload 提取 |
| 修改 | `backend/app/services/amap_parsers/adapters/http.py` | 增加 HTTP 天气与三类路线 payload 提取 |
| 修改 | `backend/app/services/amap_parsers/errors.py` | 为 `AmapParseError` 增加天气/路线 reason；继续复用 Stage 3 的 `invalid_response` error_code，不新建同义异常层 |
| 修改 | `backend/app/models/source.py` | 仅按 Stage 1 实测字段补齐天气来源模型，不改变 Stage 2 quality/freshness/null 语义 |
| 修改 | `backend/app/models/route.py` | 仅按实测能力补齐 Provider 查询方式、公交事实段和 route ID origin；不加入 Stage 10 选择结果 |
| 新增 | `backend/tests/contract/test_amap_weather_parser.py` | 使用 MCP/HTTP 脱敏 fixture 验证日期、来源、缺失和格式漂移 |
| 新增 | `backend/tests/contract/test_amap_route_parser.py` | 验证步行、驾车、公交候选、polyline、route ID、错误和通道等价语义 |
| 新增 | `backend/tests/unit/services/test_route_id.py` | 验证规范化与生成 ID 的确定性、敏感性和请求级唯一性 |
| 修改 | `backend/tests/unit/services/test_amap_service.py` | 增加天气/路线调用参数、合法空结果、错误传播与日志脱敏测试 |

旧 `backend/app/models/schemas.py`、旧 Agent、API 路由和前端本 Stage 不修改。Stage 4 完成后，新 AmapService 的 POI、地理编码、天气和路线均有结构化入口，但默认规划链仍待 Stage 5/16 接入。

## 4. 输入、输出与错误契约

### 4.1 天气服务

| 方法 | 输入 | 成功输出 | 无预报日期 |
|------|------|----------|------------|
| `get_weather` | `request_id`、非空 `city`、1～3 个连续且不重复的 `requested_dates` | 与请求日期一一对应、按日期升序的 `list[WeatherDay]` | 对应日期返回 `quality=unavailable`、`reason=outside_forecast_range` 或 `no_forecast` |

- Provider 返回的每个有效预报必须包含明确日期；同一日期重复时保留 Provider 顺序第一条并记录 duplicate 诊断。
- 昼夜天气、温度、风向和风力均为来源事实。缺失字段使用 null/unavailable，不用空字符串、`0`、气候常识或 LLM 补齐。
- 温度只接受 Stage 1 fixture 验证过的数字/带单位格式；去除受控单位后必须为合法数值。无法解析时仅该字段 unavailable，不把真实零度与缺失混淆。
- Provider 的有效预报范围之外不发明远期天气。返回列表仍覆盖全部 requested_dates，使 Stage 18 能明确识别可用和不可用日期。
- envelope 合法且预报列表为空时，全部 requested_dates 返回 unavailable/no_forecast；envelope 漂移或声称有数据但日期均非法时抛 `invalid_response`。
- 部分预报条目日期非法时跳过并记录脱敏诊断；其对应请求日期返回 unavailable/invalid_source_item。只要仍有合法预报即可返回有效子集，全部条目非法才升级为 `invalid_response`。
- `WeatherDay` 保留 provider、fetched_at、freshness 和字段级 quality；当前实时请求 freshness 为 live，缓存语义留给 Stage 17。

### 4.2 路线服务

| 方法 | 输入 | 成功输出 | 合法无路线 |
|------|------|----------|------------|
| `plan_routes` | `request_id`、不同的 GCJ-02 origin/destination、`walking/driving/transit`、策略；transit 必须提供来源要求的城市/adcode | 保持 Provider 顺序的 `list[SourcedRoute]` | `[]` |

- 新入口只接受 Stage 2 `GeoPoint`，不接受地址字符串；地址场景必须先调用 Stage 3 `geocode()` 并由上层明确选择坐标。
- `walking/driving/transit` 是 Provider 查询方式，不等同于最终用户交通选择。driving 事实可在 Stage 10A 分别用于 taxi/self_drive 语义；Stage 4 不决定其用途或费用。
- 每条路线必须包含 route ID、起终点、查询方式、distance_meters、duration_seconds、GCJ-02 polyline、provider/fetched_at/freshness 和来源策略元数据。
- distance 和 duration 必须为正数，polyline 至少包含两个合法点并保持行进顺序；任一核心字段缺失时该路线为 invalid item，不输出半真路线。
- transit 在来源提供时保留步行段、公交/地铁/旅游专线段、站点、换乘次数和线路名称；来源无法细分时使用受控 unknown，不根据名称关键词猜测交通子类。
- 驾车来源提供的 tolls、traffic_lights 或其他计费输入作为可选事实保留；Stage 4 不推导打车价格、自驾油费或停车费。
- 同一响应按 route ID 稳定去重；不同策略或不同路径的候选不得只因距离/耗时相同而合并。

### 4.3 polyline 解析

- 只接受 Stage 1 验证过的坐标串/步骤路径格式，严格按 longitude、latitude 解析为 GCJ-02 `GeoPoint`。
- 多步骤路径按来源顺序拼接，仅删除步骤边界或相邻的完全重复点；不做抽稀、平滑、直线补点、坐标平移或缺段猜补。
- 非数字、数量错误、越界、坐标系不明或少于两个有效点均使该路线 invalid；禁止退化为起终点直线。
- 请求 origin/destination 与 Provider 返回的道路吸附起终点（若有）必须作为独立字段保留；polyline 只校验来源顺序和合法坐标，不因道路吸附造成的首尾距离单独拒绝真实路线，也不得人为插入直线连接。Stage 1 ADR 记录观察到的首尾差异，Stage 20 展示时使用真实 polyline 与独立点位标记。

### 4.4 route ID

- 优先使用 Provider 原生稳定 route ID，并标记 `route_id_origin=source`。
- 无原生 ID 时，将 provider、规范化起终点、查询方式、策略、distance、duration 和规范化 polyline 摘要按固定字段顺序编码为 UTF-8 canonical JSON，计算 SHA-256，取前 24 个十六进制字符并加 `gen_` 前缀，标记 `route_id_origin=generated`。
- 起终点和 polyline 坐标按 Stage 2 序列化精度规范化；禁止使用 Python `hash()`、数组下标、当前时间或随机值。
- 生成 ID 只承诺同一规范化候选内容和当前请求候选集合内稳定；路线事实变化必须产生不同 ID，不承诺跨 Provider 等价路线 ID 相同。
- `route_id_origin=generated` 只描述标识符生成方式，路线距离、耗时和 polyline 的 DataQuality 仍为 sourced，不得把整条真实路线误标为 generated。

### 4.5 空结果、坏条目与错误

- 合法路线空集合返回 `[]`；部分路线非法时返回合法子集并记录脱敏诊断；声称有路线但全部核心字段非法时抛 `invalid_response`。
- 天气按请求日期返回 unavailable 占位，不用空列表表达超出预报范围；只有请求本身非法才是校验错误。
- Stage 1 ProviderError 六类原样传播，不转换为空结果；解析层沿用 Stage 3 `AmapParseError(error_code="invalid_response")`。
- 错误/诊断最多包含 request_id、provider、operation、mode、条目序号和稳定 reason/字段名；不得包含 Key、完整 URL、完整 payload、完整公交站点列表或用户自由文本。
- 本 Stage 不映射 HTTP 状态码；Stage 15 统一映射。Stage 5 shadow 必须能区分业务空路线、天气 unavailable 和技术错误。

### 4.6 下游职责边界

- Stage 4 只提供真实路线候选，不选择偏好方式、不执行 walking 2km 淘汰、不做 taxi/transit 回退排序；这些属于 Stage 10A/10B。
- Stage 4 不把天气转为室内外标签，不替换景点、不重建草案；这些属于 Stage 18。
- Stage 4 不计算路线起止时间、缓冲或时间冲突；这些属于 Stage 11/12B。
- Stage 4 不决定地图展示样式或导出路线快照；这些属于 Stage 20/25。

## 5. 配置与功能开关

- 本 Stage 不新增环境变量或用户流量开关；沿用 Stage 1 Provider 配置、factory 和能力矩阵。
- 路线策略允许值、transit 必需城市参数和道路吸附字段能力来自 Stage 1 ADR，固化为 Provider capability/代码常量，不开放任意环境覆盖。
- AmapService 继续由装配层注入 Provider 和时钟；解析器不读取 Settings。
- 不新增路线偏好、walking 上限、缓存 TTL、重试、并发或地图 Key 配置。

## 6. 实施步骤

1. 依据 Stage 1 ADR 固定天气与三类路线的 Provider payload 适配入口、策略值和能力字段，并补齐最小脱敏 fixture 元数据。
2. 实现天气 envelope、日期和字段级解析，按 requested_dates 生成 sourced/unavailable 一一对应结果。
3. 实现共享 polyline 解析与步骤拼接，校验坐标、点数和顺序，独立保留请求点与来源道路吸附点，不插入连接直线。
4. 实现三类路线 envelope/逐条解析，保留策略、公交段和可选驾车事实，区分空结果、坏条目和 invalid_response。
5. 实现原生/生成 route ID；用 canonical JSON 和 SHA-256 锁定确定性，不依赖进程状态。
6. 扩展注入式 AmapService 的 `get_weather()` 与 `plan_routes()`，传播 ProviderError/AmapParseError，不触碰 Stage 3 已通过的 POI/地理编码行为。
7. 完成双通道 fixture 契约测试、FakeAmapProvider 服务测试和日志脱敏检查，确认 Stage 10/18/20 职责未被提前实现。
8. 执行 Stage 1～4 定向测试、后端全量回归和前端构建，归档字段、错误和回滚证据。

## 7. 明确不做

- 不实现路线候选选择、交通偏好回退、walking 2km 过滤、混合交通决策或费用计算。
- 不实现天气驱动候选筛选、室内外推断、草案重排或行前建议。
- 不实现 TimelineCalculator/Validator、地图联动、SSE、缓存、stale 降级、并发和自动重试。
- 不支持新入口直接传地址，不在路线解析器内部偷偷地理编码。
- 不改旧 Agent、API 路由、前端类型/页面或默认 `/trip/plan`，不删除旧 `schemas.py`。
- 不以空字典、默认天气、起终点直线或 LLM 文本作为结构化成功结果。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S4-T01 | contract | MCP/HTTP 天气成功 fixture | 同日期输出相同领域语义，来源通道不改变 quality 规则 |
| S4-T02 | contract | 1～3 个 requested_dates 与乱序/重复预报 | 输出按日期一一对应；重复日期保留第一条并记录诊断 |
| S4-T03 | contract | 预报范围外与合法空预报 | 对应日期为 unavailable/outside_forecast_range 或 no_forecast，不生成默认天气 |
| S4-T04 | contract | 零度、缺失温度和非法温度字符串 | 真实 0 保留；缺失/非法为字段级 unavailable，不回退 0 |
| S4-T05 | contract | 部分日期非法、全部日期非法或 envelope 漂移 | 部分非法返回有效子集 + unavailable/invalid_source_item；全部非法/漂移抛 invalid_response |
| S4-T06 | contract | 步行、驾车、公交成功 fixture | 距离、耗时、方式、策略、polyline 与来源元数据正确 |
| S4-T07 | contract | 合法空路线、部分坏路线、全部坏路线 | 分别返回 `[]`、合法子集 + 诊断、invalid_response |
| S4-T08 | unit | polyline 单段、多步骤和相邻重复点 | 顺序拼接，只删除相邻完全重复点，不抽稀或补直线 |
| S4-T09 | unit | polyline 非数字、越界、坐标系不明、少于两点 | 路线拒绝且 reason 稳定 |
| S4-T10 | contract | 请求点、道路吸附点与 polyline 首尾 | 三者独立保留；道路吸附差异不拒绝真实路线，也不插入连接直线 |
| S4-T11 | contract | 公交方式细分完整/缺失 | 来源事实保留；无法细分为 unknown，不按名称猜测 |
| S4-T12 | contract | 驾车可选计费字段缺失 | 核心路线仍有效；可选字段 null/unavailable，不推导费用 |
| S4-T13 | unit | 原生 route ID 与生成 route ID | 原生优先；同规范输入 ID 相同，路线摘要变化 ID 变化，格式固定 |
| S4-T14 | unit | 无效 mode、相同起终点、transit 缺城市 | 输入校验失败，不静默回退 walking |
| S4-T15 | unit | FakeAmapProvider 六类错误 | 分类原样传播，不返回空集合/空字典 |
| S4-T16 | unit | 注入时钟、诊断和日志 | 时间可重复；日志无 Key、完整 payload/URL/站点列表 |
| S4-T17 | contract | Stage 3 与下游边界回归 | POI/地理编码不变；无路线选择、天气重排或地图展示逻辑 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/contract/test_amap_weather_parser.py backend/tests/contract/test_amap_route_parser.py backend/tests/unit/services/test_route_id.py backend/tests/unit/services/test_amap_service.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网或真实高德服务。Stage 4 必测用例数为 17，失败数必须为 0；外部能力和误差阈值只引用 Stage 1 实测证据。

## 9. 人工验收

不适用：Stage 4 不接入 API/UI，天气、路线、错误、ID 和日志行为均由 Stage 1 脱敏 fixture 与 FakeAmapProvider 覆盖。评审者只核对 `field-matrix.md` 与 Stage 1 ADR 的能力/误差引用，不重新消耗真实配额。

## 10. 量化退出指标

- [ ] S4-T01～S4-T17 全部通过，失败数为 0；三条标准命令退出码均为 `0`。
- [ ] 请求的 1～3 个天气日期均有对应结果；预报外/空预报为 unavailable，真实零度不与缺失混淆，非法 envelope 不冒充 unavailable。
- [ ] walking/driving/transit 的每条输出路线均有真实距离、耗时、GCJ-02 polyline、策略、来源和稳定 route ID。
- [ ] 合法空路线、部分坏路线、整体非法响应和 ProviderError 四类结果互不冒充；路线失败不再返回 `{}`。
- [ ] polyline 不抽稀、不补直线、不平移；非法坐标/未知坐标系/少于两点稳定拒绝；请求点、道路吸附点和路径首尾不会互相覆盖。
- [ ] 原生 route ID 优先；生成 ID 使用 canonical JSON + SHA-256，固定输入重复结果一致，事实变化结果不同。
- [ ] 公交子类只保留来源事实，驾车可选字段不被推导为费用；Stage 4 不产生路线选择结果。
- [ ] 默认测试公网请求数为 0；日志、诊断、异常和证据中的 Key、完整 URL/payload/站点列表命中数为 0。
- [ ] Stage 3 全部回归通过；旧 `/trip/plan` 与前端构建不受影响，新增未追踪 TODO 数为 0。
- [ ] 03 Stage 4 的每条退出条件以及 02 的天气/路线真实性规则均映射到测试或证据 ID。

## 11. 迁移与回滚

- Stage 4 新入口不接默认流量；回滚恢复 `amap_service.py` 到 Stage 3 结构，删除天气/路线解析器及对应测试，不涉及 TripPlan、前端、缓存或持久化迁移。
- `models/source.py` 和 `models/route.py` 只允许增量兼容字段；后续 Stage 已依赖时必须按依赖逆序回滚。
- 回滚后 Stage 1～3 测试、旧模型导入检查和前端生产构建必须通过；天气/路线旧空结果行为只存在于未切换的 legacy 链路。
- Stage 1 的天气/路线能力、误差和失败 fixture 属于决策证据，不随代码回滚删除。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_4/test-summary.md` | 提交标识、三条命令、17 个用例结果、离线声明和失败数 |
| 字段矩阵 | `doc/opt_strategy/evidence/stage_4/field-matrix.md` | Provider 天气/路线字段 → parser → Stage 2 模型 → quality/reason → 测试 ID |
| route ID 说明 | `doc/opt_strategy/evidence/stage_4/route-id-summary.md` | 原生/生成规则、canonical 字段、摘要变化样本和稳定性结论 |
| 错误矩阵 | `doc/opt_strategy/evidence/stage_4/error-matrix.md` | 空结果、unavailable、坏条目、invalid_response 和 ProviderError 行为 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_4/rollback-check.md` | 回滚范围、依赖逆序要求、验证命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计复核结论：**R2～R10 通过首轮跨 Stage 审计；执行 Ready 待 R1**。R1 必须等待 Stage 1～3 实际 Done，并在 Stage 4 开工前再次确认 Provider、来源模型、适配器和 AmapService 契约未变化；当前不表示 Ready、Done、设计冻结或允许修改工程代码。
