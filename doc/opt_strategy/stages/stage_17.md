# Stage 17：进程内请求缓存与外部数据降级

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 17 |
| 目标 | 为 AmapService/RouteService 使用的外部事实读取增加单进程有界 TTL 缓存，减少重复调用，并在实时 Provider 失败时仅以可信 stale 数据形成可追溯降级 |
| 对应优化项 | P1-3 请求缓存与外部数据降级，并补齐 P1-1/P1-2 的真实 `degraded=true` 输入来源 |
| 前置依赖 | Stage 1、16；开工前复核 Stage 2～4 的 SourceMetadata/Freshness、Stage 10A/10B 的请求内去重、Stage 15 错误 envelope 与 Stage 16 正式 TripPlan |
| 后续阻断 | 未通过时不得宣告 Gate C；不得开始依赖缓存语义的 Stage 20、23、28、30，也不得把 stale 降级作为已上线能力写入验收 |
| 默认流量影响 | Stage 16 已默认 new 后，普通请求可使用缓存；不改变 API schema、Provider 选择、规划顺序或前端状态，只通过来源 metadata 影响 `degraded` 提示 |

## 2. 当前代码基线与差异

- 当前 `backend/app/services/amap_service.py` 直接调用 MCPTool，异常时返回空列表/空字典/None；没有结构化 Provider、来源 freshness、缓存键、TTL、容量、stale 或错误分流。
- Stage 3/4 目标 AmapService 将在 Provider 注入、结构化解析和 `SourceMetadata(freshness=live)` 基础上输出 `SourcedPOI`、`GeocodeCandidate`、`WeatherDay`、`SourcedRoute`；Stage 10A/10B 仅有请求内去重，不提供跨请求缓存。
- Stage 15 已规定技术错误在无可信输入时返回 error envelope，Stage 16 已规定 `degraded=true` 只能由可信 stale 来源传入；但真实缓存/stale 机制尚未存在。
- 04 总纲已登记 `TRIP_CACHE_MAX_ENTRIES`、`TRIP_CACHE_TTL_POI_SECONDS`、`TRIP_CACHE_TTL_WEATHER_SECONDS`、`TRIP_CACHE_TTL_ROUTE_SECONDS`，具体默认值、容量和淘汰策略必须引用 Stage 1 ADR/实测结论定稿。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 新增 | `backend/app/services/cache.py` | 实现注入时钟的单进程有界 TTL 缓存、LRU/过期清理、fresh/stale 查询与统计 | 不依赖 Redis/磁盘/全局时间；容量、TTL、淘汰可测 |
| 新增 | `backend/app/services/cache_keys.py` | 为 POI、geocode、weather、route 构造规范化缓存键，包含操作、参数、Provider、数据版本和策略 | 等价请求同键，语义不同请求不同键；不含 request_id/Key/自由文本 |
| 修改 | `backend/app/models/source.py`、`models/__init__.py` | 固化 `Freshness.cached_fresh/cached_stale` 与缓存元数据字段；保持 `DataQuality` 不变 | cached/stale 只表达时效，不改变 sourced/generated 等质量语义 |
| 修改 | `backend/app/services/amap_service.py` | 在结构化入口外包缓存读写与 stale 降级；Provider 成功写缓存，fresh 命中跳过 Provider，实时失败时尝试 stale | Provider 技术错误、合法空结果和 stale 行为互不冒充 |
| 修改 | `backend/app/services/route_service.py` | 保留 10A/10B 请求内去重优先级，同时让底层 AmapService 路线查询可跨请求缓存 | 同一请求仍不重复调用；跨请求可命中缓存 |
| 修改 | `backend/app/coordinators/trip_coordinator.py`、`services/trip_assembler.py` | 汇总输入事实 freshness，任一正式使用 stale 时设置 `TripPlan.degraded=true` 与脱敏降级摘要 | complete/partial/failed 均可 degraded；无 stale 时 false |
| 修改 | `backend/app/api/routes/trip.py`、`api/error_mapping.py` | 保持 Stage 15/16 响应语义；无 fresh/stale 可信输入时继续返回对应技术错误 | 不用 failed/partial 掩盖技术故障 |
| 修改 | `backend/app/config.py`、`backend/.env.example` | 接入缓存容量与 POI/天气/路线 TTL 配置；缺失使用 Stage 1 ADR 定稿默认值，非法值启动失败 | 启动日志只打印是否启用和数值范围，不含 Key |
| 新增 | `backend/tests/unit/services/test_request_cache.py` | 覆盖 TTL、容量、LRU、时钟、key、fresh/stale/过期行为 | 离线、无真实 Provider |
| 新增 | `backend/tests/unit/services/test_amap_service_cache.py` | 用 FakeAmapProvider 覆盖有效命中、过期重取、实时失败转 stale、无 stale 失败 | 四类核心场景稳定 |
| 新增 | `backend/tests/integration/test_cache_degraded_flow.py` | 覆盖 Coordinator/TripPlan `degraded`、错误分流和日志脱敏 | stale 成功为 degraded，坏 stale/无 stale 为 error |
| 新增 | `doc/opt_strategy/evidence/stage_17/` | 记录 ADR 引用、缓存矩阵、测试摘要、人工/回滚证据 | 不提交完整缓存值、payload、Key 或自由文本 |

不新增前端文件；Stage 16 已具备 degraded 展示语义，本 Stage 只提供真实来源输入。

## 4. 输入、输出与缓存契约

### 4.1 缓存适用范围

| 操作 | 可缓存输出 | TTL 配置 | stale 允许 | 特别约束 |
|---|---|---|---|---|
| `search_poi` | `list[SourcedPOI]`，含合法空列表 | `TRIP_CACHE_TTL_POI_SECONDS` | 是 | key 必含 keywords/type/city/city_limit/page_size/Provider/数据版本 |
| `geocode` | `list[GeocodeCandidate]`，含合法空列表 | `TRIP_CACHE_TTL_POI_SECONDS` | 是 | key 必含 address/city/Provider/坐标系版本；不得缓存非法地址输入 |
| `get_weather` | `list[WeatherDay]`，含 unavailable 日期 | `TRIP_CACHE_TTL_WEATHER_SECONDS` | 是 | key 必含 city/adcode/requested_dates/Provider/预报模式 |
| `plan_routes` | `list[SourcedRoute]`，含合法空列表 | `TRIP_CACHE_TTL_ROUTE_SECONDS` | 是 | key 必含 origin/destination GCJ-02、query_mode、strategy、transit city/adcode、departure_at bucket |

- 仅结构化、已通过 Stage 3/4 parser 校验的领域对象可以入缓存；Provider raw payload、Prompt、完整自由文本、API Key、异常对象和未脱敏诊断不得入缓存。
- 合法空结果可以 fresh 缓存，代表 Provider 明确返回空集合；它不是技术错误，也不是 stale。实时失败后若 stale 中只有合法空结果，仍可按业务不可行继续，但必须标记 `cached_stale`。
- 业务层不得直接访问缓存实例；只通过 AmapService 的统一结构化方法消费来源对象。

### 4.2 读取与写入顺序

1. AmapService 根据规范化参数生成 cache key；key 不包含 `request_id`、完整自由文本、用户原始表单、Key、token 或不可稳定排序的字典。
2. 查询 fresh 缓存：命中时返回深拷贝/不可变副本，并将所有来源对象 freshness 改为 `cached_fresh`，Provider 调用次数为 0。
3. 未命中 fresh 时同步串行调用当前已选 Provider；Provider 成功且 parser 校验通过后写入缓存并返回 `live`。
4. Provider 发生 Stage 1/3/4 的技术错误时，才允许查询 stale；存在可校验 stale 时返回 `cached_stale` 并记录降级摘要。
5. 无 stale、stale schema/version 不兼容、stale 值无法通过当前领域模型校验或缓存读取自身异常时，原 Provider 技术错误继续交 Stage 15 映射。

### 4.3 stale 与 degraded 传播

- `cached_stale` 是唯一触发本 Stage `degraded=true` 的缓存来源；`cached_fresh` 不触发 degraded。
- TripCoordinator 汇总所有被正式使用的外部事实来源：任一 POI/酒店/餐饮/天气/路线/地理编码为 `cached_stale`，则最终 complete/partial/failed 业务结果设置 `degraded=true`，并在内部 diagnostics 记录 operation、key hash、stale age bucket 和原因码。
- `degraded` 不改变 `plan_status`，不放宽 accepted constraints、路线真实性、时间轴、预算或 partial/failed 规则。
- 若 stale 只能覆盖部分必要输入，其余必要输入无 fresh/live/stale，则不得构造半真半假的计划；按 Stage 15 技术错误或 Stage 13D 业务不可行分流。

### 4.4 缓存键、版本和 Provider 边界

- key 必须包含 `schema_version`、`operation`、`provider_name`、`provider_capability_version`、`coordinate_system=gcj02`、标准化参数和影响语义的策略字段。
- MCP 与官方 HTTP Provider 输出即使领域模型相同，也不得默认共用缓存条目；只有后续 ADR 明确证明兼容并升级 key 版本后才可合并。
- Provider 人工切换不是数据降级，不设置 `degraded=true`；切换后因 key 不同自然避免误用旧 Provider 的不兼容条目。
- 过期判断使用注入时钟；stale 最大可用窗口由 Stage 1 ADR 定稿并写入证据，超过窗口的条目必须视为不可用并可被清理。

## 5. 配置与功能开关

| 配置 | 类型 | 默认与来源 | 非法行为 | 备注 |
|---|---|---|---|---|
| `TRIP_CACHE_MAX_ENTRIES` | 正整数 | Stage 1 ADR 定稿默认值，可 env 覆盖 | 启动失败，映射配置错误 | 单进程总容量，按条目计 |
| `TRIP_CACHE_TTL_POI_SECONDS` | 正整数 | Stage 1 ADR 定稿默认值，可 env 覆盖 | 启动失败，映射配置错误 | POI 与 geocode 共用 |
| `TRIP_CACHE_TTL_WEATHER_SECONDS` | 正整数 | Stage 1 ADR 定稿默认值，可 env 覆盖 | 启动失败，映射配置错误 | 天气预报不得超过 Provider 可用语义 |
| `TRIP_CACHE_TTL_ROUTE_SECONDS` | 正整数 | Stage 1 ADR 定稿默认值，可 env 覆盖 | 启动失败，映射配置错误 | route key 已含策略与出发参考 |

不新增 `TRIP_CACHE_ENABLED` 开关。回滚通过撤销 Stage 17 接入或将容量/TTL 恢复到工作包指定默认值完成；不得引入“缓存关闭但 stale 仍生效”的双语义。

## 6. 实施步骤

1. 从 Stage 1 ADR 抽取 Provider 稳定性、字段能力、TTL/容量建议和 stale 最大窗口，形成 `cache-adr.md` 证据；若 ADR 缺失数值，本 Stage 先阻断实施，不臆造生产式默认。
2. 实现纯内存 `RequestCache`、`CacheEntry`、注入时钟、容量淘汰、过期清理和 key hash 日志工具；所有值写入前后均经领域模型校验。
3. 为 POI、geocode、weather、route 增加确定性 key builder 和等价/非等价参数测试，锁定 Provider、schema、坐标系和策略边界。
4. 在 AmapService 结构化入口接入 read-through/write-through/stale fallback；保持 ProviderError/AmapParseError 原分类，不用缓存异常覆盖主错误。
5. 让 Coordinator/Assembler 汇总 freshness 并设置 `degraded` 与脱敏降级摘要；Stage 16 前端无需新增字段。
6. 用 FakeAmapProvider 和注入时钟完成定向、集成、全量后端和前端 build；归档缓存命中、stale、错误、配置和回滚证据。

## 7. 明确不做

- 不引入 Redis、SQLite、磁盘缓存、跨进程共享、后台刷新、异步并发、限流器、自动 Provider 切换或预热任务。
- 不改变 Provider 选择、API schema、TripPlan status、修复/partial 规则、天气决策、地图展示、图片批量接口、SSE 或前端布局。
- 不缓存 LLM 输出、Agent 草案、完整 TripPlan、用户完整请求、自由文本、Prompt、Provider raw payload、错误 envelope 或 PDF/图片二进制。
- 不以 stale 数据补写缺失事实；stale 只能复用曾经通过校验的来源对象。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S17-T01 | unit | TTL 与注入时钟 | fresh、stale、不可用窗口判定稳定 |
| S17-T02 | unit | 容量与 LRU/过期清理 | 超容量淘汰可预测；清理不影响 fresh 命中 |
| S17-T03 | unit | cache key 等价性 | 参数顺序不影响 key；Provider/策略/schema 变化必不同 key |
| S17-T04 | unit | fresh 命中 | 返回 `cached_fresh`，Provider 调用次数为 0 |
| S17-T05 | unit | 过期后实时成功 | 调用 Provider，返回 `live` 并刷新缓存 |
| S17-T06 | unit | 实时失败转 stale | 返回 `cached_stale`，保留原 Provider 错误为内部 cause，触发 degraded |
| S17-T07 | unit | 实时失败且无 stale/坏 stale | 原技术错误进入 Stage 15 映射，不构造空结果或 failed plan |
| S17-T08 | unit | 合法空结果缓存 | fresh 空集合命中不调用 Provider；stale 空集合只表达旧空结果 |
| S17-T09 | integration | POI/geocode/weather/route 四类操作 | key、TTL、freshness 和 Provider 调用次数均符合表格 |
| S17-T10 | integration | Coordinator degraded 汇总 | 任一正式使用 stale 时 TripPlan `degraded=true`，无 stale 时 false |
| S17-T11 | api | 技术错误无可信输入 | 返回 Stage 15 error envelope，无 TripPlan |
| S17-T12 | regression | Stage 10A/10B 请求内去重 | 同请求重复段仍只查询一次；跨请求可命中 cache |
| S17-T13 | config | 配置缺失/非法/日志 | 默认值来自 ADR，非法启动失败，日志无 Key/payload |
| S17-T14 | security | 缓存值与证据脱敏 | 无 raw payload、Prompt、自由文本、Key、完整地址列表 |

```text
python -m pytest backend/tests/unit/services/test_request_cache.py backend/tests/unit/services/test_amap_service_cache.py backend/tests/integration/test_cache_degraded_flow.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S17-M01 | 使用 fake Provider 连续提交同一结构化请求 | 第二次命中 fresh，Provider 调用计数不增加，页面/响应 schema 不变 |
| S17-M02 | 让缓存过期后注入 Provider timeout 且存在 stale | 返回正常业务结果并显示/记录 degraded；不暴露 Provider 原始错误 |
| S17-M03 | 注入 Provider timeout 且无 stale | 返回稳定技术错误 envelope；不生成 partial/failed 或虚构事实 |
| S17-M04 | 切换 Provider 或修改路线策略后重复请求 | 不复用旧 key；degraded 不因 Provider 切换本身出现 |

## 10. 量化退出指标

- [ ] S17-T01～T14、S17-M01～M04 通过，三条命令退出码为 0。
- [ ] fresh 命中场景 Provider 调用次数为 0；过期实时成功场景 Provider 调用次数为 1；同请求重复路线外部调用数仍为 0。
- [ ] 技术失败有可信 stale 时 `cached_stale/degraded=true` 正确率 100%；无可信 stale 时技术错误被伪装成业务结果的次数为 0。
- [ ] 缓存 key 中 request_id、Key、Prompt、完整自由文本和完整 payload 命中数为 0；Provider/schema/策略变化误命中数为 0。
- [ ] 缓存读写/淘汰异常影响实时主链路的次数为 0；缓存自身异常只产生脱敏诊断，不改变 Provider 成功结果。

## 11. 迁移与回滚

- 迁移顺序：配置与 ADR 证据 → cache/key 单元 → AmapService 接入 → degraded 汇总 → 集成/API 证据；不得先让 Coordinator 消费 stale 再补 key/schema 校验。
- Stage 17 只引入内存状态，进程重启可丢失；无数据迁移、无持久化清理、无前端 storage 变更。
- 回滚先移除 Coordinator degraded 汇总中的 cache 来源消费，再移除 AmapService 缓存包装，最后删除 cache/key 模块和配置；保留 Stage 1～16 的 Provider、错误和正式结果语义。
- 回滚后运行 S17-T04～T07、S17-T10～T11、全量后端和前端 build，并记录 Provider 调用次数、degraded=false 回归和无虚构 fallback。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| ADR/默认值 | `doc/opt_strategy/evidence/stage_17/cache-adr.md` | Stage 1 实测引用、TTL、容量、stale 窗口、Provider 边界 |
| 自动化摘要 | `doc/opt_strategy/evidence/stage_17/test-summary.md` | 命令、14 用例、离线声明、失败数 |
| 缓存键矩阵 | `doc/opt_strategy/evidence/stage_17/cache-key-matrix.md` | operation、参数、Provider/schema/策略、等价/非等价样例 |
| Freshness 矩阵 | `doc/opt_strategy/evidence/stage_17/freshness-degraded-matrix.md` | live/cached_fresh/cached_stale、degraded、错误分流与测试 ID |
| 调用次数矩阵 | `doc/opt_strategy/evidence/stage_17/provider-call-count.md` | fresh、expired、stale、route de-dup、Provider 切换计数 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_17/manual-check.md`、`rollback-check.md` | M01～M04、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 17 独立设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 1 与 Stage 16 实际 Done，并在开工前复核 Stage 1 ADR 的 TTL/容量/stale 窗口、Stage 2 SourceMetadata/Freshness、Stage 3/4 AmapService 结构化入口、Stage 10A/10B RouteService 去重、Stage 15 错误 envelope 和 Stage 16 `degraded` 正式 schema。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
