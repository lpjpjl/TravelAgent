# Stage 1：高德 MCP 与官方 HTTP Provider 技术验证
> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|------|------|
| Stage | 1 |
| 目标 | 用可重复的脱敏样本和受控 live verification 消除 Provider 选择、字段完整性、错误形态、串行稳定性与配额约束的不确定性 |
| 对应优化项 | P0-1、P0-6、P1-3 的前置验证 |
| 前置依赖 | 无 |
| 后续阻断 | 未通过时不得开始 Stage 2、3、4、10A、10B、17 |
| 默认流量影响 | 无；不替换当前 `/trip/plan` 链路 |

## 2. 当前代码基线与差异

- 当前入口为 `backend/app/services/amap_service.py` 的 `get_amap_mcp_tool()` 与 `AmapService`。服务在构造时直接创建 MCP 工具，没有可替换的 Provider 边界。
- 当前 MCP 命令固定为 `uvx amap-mcp-server`，工具名散落在服务方法中；目标版本、工具 schema 和返回层级没有被仓库内契约固定。
- `search_poi()`、`get_weather()`、`plan_route()` 和 `geocode()` 只截取打印原始结果，随后返回空列表、空字典或 `None`；异常同样被吞并为空结果，无法区分合法空结果、上游错误与格式漂移。
- `backend/app/agents/trip_planner_agent.py` 另行创建 MCPTool，尚未复用 AmapService。Stage 1 不迁移旧 Agent，只记录这一重复入口，留待 Stage 5/16 收口。
- `backend/app/config.py` 只有 `amap_api_key`，尚无 `amap_provider`；`backend/requirements.txt` 尚无 pytest，仓库也没有 `backend/tests/`。
- 当前没有官方 HTTP Provider、脱敏高德 fixture、Provider 契约测试或 live verification 入口。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 |
|------|------|----------------|
| 新增 | `backend/app/providers/amap/base.py` | 定义最小 Provider 协议、原始 DTO 包装和统一 Provider 异常，不定义 Stage 2 领域模型 |
| 新增 | `backend/app/providers/amap/mcp.py` | 封装 MCP 工具发现与四类能力调用；保留原始响应边界 |
| 新增 | `backend/app/providers/amap/http.py` | 封装官方 HTTP 对等接口，用于受控验证；不做运行期自动故障切换 |
| 新增 | `backend/app/providers/amap/factory.py` | 依据 Settings 显式创建单一 Provider，拒绝未知名称 |
| 新增 | `backend/app/providers/amap/__init__.py` | 仅导出稳定的 Provider 创建入口和公共类型 |
| 修改 | `backend/app/config.py` | 增加 `amap_provider` 配置读取与枚举校验；Key 继续只从 Settings 注入 |
| 修改 | `backend/requirements.txt` | 增加最小 pytest 依赖，不引入覆盖率、并行或网络测试插件 |
| 新增 | `backend/tests/conftest.py` | 提供 fixture 根路径等最小公共测试支持，不加载真实 `.env` Key |
| 新增 | `backend/tests/fixtures/amap/` | 保存 MCP/HTTP 的成功、空结果、业务错误、限流、鉴权失败和格式漂移脱敏样本及元数据 |
| 新增 | `backend/tests/contract/test_amap_provider_contract.py` | 离线验证两种 Provider 的统一方法、原始 DTO 包装和错误分类 |
| 新增 | `backend/tests/unit/test_amap_provider_factory.py` | 验证 Provider 配置、factory、缺失 Key 和日志脱敏 |
| 新增 | `backend/tests/live/verify_amap_provider.py` | 显式人工运行的受控验证入口；不被 pytest 默认收集 |
| 新增 | `doc/opt_strategy/decisions/amap_provider_validation.md` | 记录能力矩阵、实测数据、配额来源、默认 Provider 决策与未关闭风险 |
| 新增 | `doc/opt_strategy/evidence/stage_1/` | 保存脱敏测试、人工验证与回滚摘要 |

`backend/app/services/amap_service.py` 本 Stage 不切换到新 Provider；Stage 3、4 才分别接入结构化 POI/地理编码与天气/路线解析，避免技术验证和业务迁移混在一个变更集。

## 4. 输入、输出与错误契约

最小 `AmapProvider` 只覆盖以下同步方法；参数在 Provider 边界统一，MCP/HTTP 特有参数不得泄漏给调用方：

| 方法 | 最小输入 | Stage 1 输出 |
|------|----------|--------------|
| `search_poi` | `request_id`、`keywords`、`city`、`city_limit` | `ProviderRawResult`，含 provider、operation、原始坐标系和运行期完整可解析 payload |
| `geocode` | `request_id`、`address`、可选 `city` | 同上 |
| `get_weather` | `request_id`、`city` | 同上 |
| `plan_route` | `request_id`、起终点坐标、`walking/driving/transit`、公交所需城市 | 同上 |

- `ProviderRawResult` 不进入 API，也不承诺最终领域字段；它只固定通道、操作名、payload 和采集元数据边界，为 Stage 2～4 的模型与解析提供输入。运行期 payload 保留解析所需完整字段；只有提交仓库的 fixture 和证据执行最小化脱敏，二者不得混为同一对象。
- `request_id` 是调用上下文而非上游业务参数：必须贯穿日志和错误，但不得拼入高德查询条件、缓存键或返回 payload。
- MCP 与 HTTP Provider 的方法签名和异常类别必须一致，但原始 payload 允许不同；禁止为追求 JSON 全文一致而丢弃来源字段。
- 统一异常最少区分：`configuration_error`、`authentication_error`、`rate_limited`、`timeout`、`upstream_error`、`invalid_response`。合法空结果必须作为成功响应保留，不得映射为异常。
- Provider 不捕获后返回空容器；异常必须保留稳定类别、`retryable` 和脱敏上下文，禁止包含 Key、完整 URL 查询串或未脱敏响应。
- 本 Stage 只验证字段可得性。POI、天气和路线领域对象、营业时间规范化及 GCJ-02 最终校验分别在 Stage 2～4 定义和实现。

## 5. 配置与功能开关

| Settings 字段 | 环境变量 | 允许值/行为 |
|---------------|----------|-------------|
| `amap_provider` | `AMAP_PROVIDER` | `mcp` 或 `http`；默认值只能在 ADR 根据实测结论写定，结论形成前 live 脚本必须显式传入 |
| `amap_api_key` | `AMAP_API_KEY` | secret；目标 Provider 被创建但 Key 缺失时快速失败，错误与日志只显示“未配置” |

- 不新增用户流量功能开关；验证入口与默认应用入口隔离。
- 不在本 Stage 臆造超时、重试、配额或 TTL 数值。实测值和官方限制进入 ADR，运行期策略由后续使用 Stage 定稿。
- Provider 在一次进程生命周期中显式选定；单次失败不得暗中切换另一 Provider。

## 6. 实现步骤

1. 引入最小 pytest 工具链和测试目录，先建立一个不访问公网、不读取真实 Key 的空基线测试。
2. 定义 Provider 协议、原始结果包装与异常分类，并用 fake payload 完成离线契约测试。
3. 将当前 MCP 启动、工具发现和调用封装进 MCP Provider；记录实际工具名与 schema，不修改旧 Agent 调用链。
4. 实现最小 HTTP Provider，对齐 POI、地理编码、天气、步行、驾车和公交接口；HTTP 请求日志必须去除 Key 与完整查询串。
5. 对两种通道分别采集最小脱敏 fixture，覆盖成功、空结果、鉴权、限流、超时替身、上游错误和字段漂移。
6. 使用固定的小规模样本执行 live verification：同城 POI/餐饮、地理编码、天气，以及步行/驾车/公交路线；记录字段可得性和地址调用与坐标调用差异。
7. 对选定样本执行同步串行连续调用，记录调用数、每次耗时、总耗时、失败类别以及 MCP 子进程/会话是否释放；禁止用并发压测替代本阶段串行目标。
8. 根据实测填写 ADR 的 Provider 能力矩阵、1～3 日调用预算估算、官方配额/限流来源和默认 Provider 决策。
9. 固化所选 Provider 的成功与异常 fixture，执行离线契约测试和前端构建回归，形成 Stage 1 证据摘要。

## 7. 明确不做

- 不实现 Stage 2 的领域 schema，不在本 Stage 确定 TripPlan、DataQuality 或来源状态模型。
- 不解析并返回最终 POI、天气、路线对象，不消除 `AmapService` 中的 TODO。
- 不修改 `/trip/plan`、前端页面或当前 Agent Prompt，不移除旧 MCPTool 实例。
- 不实现缓存、重试策略、运行期 Provider 自动切换、并发查询、SSE 或生产部署配置。
- 不提交真实 Key、完整上游响应、完整请求 URL、压力测试大日志或未经脱敏的用户输入。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---------|------|------|------|
| S1-T01 | contract | MCP/HTTP 实现满足同一 Provider 协议 | 四类方法可由统一调用方调用，request_id 可传播且 Provider 特有参数不外泄 |
| S1-T02 | contract | 成功 fixture | 返回带正确 provider、operation 和 payload 的原始结果包装 |
| S1-T03 | contract | 合法空结果 fixture | 返回成功空结果，不抛异常、不伪造实体 |
| S1-T04 | contract | 鉴权失败与限流 fixture | 分别映射稳定异常类别，`retryable` 符合 ADR 结论 |
| S1-T05 | contract | 超时、上游错误与非法响应 fixture | 不返回空容器，错误不包含 secret 或完整响应 |
| S1-T06 | unit | `AMAP_PROVIDER` 合法、非法、缺失 | 合法值可创建对应 Provider；非法值快速失败；缺失按 ADR 确定的默认值处理 |
| S1-T07 | unit | `AMAP_API_KEY` 缺失及异常文本 | Provider 创建/首次调用快速失败，输出不含 Key 值 |
| S1-T08 | contract | fixture 元数据检查 | 每个 fixture 声明通道、采集日期、场景和脱敏方式 |

验收命令从仓库根目录执行：

```text
python -m pytest backend/tests/contract/test_amap_provider_contract.py backend/tests/unit/test_amap_provider_factory.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，且均不得访问公网。live verification 不属于默认 pytest 集合。

## 9. 人工验收

| 用例 ID | 步骤 | 预期记录 |
|---------|------|----------|
| S1-M01 | 对 MCP/HTTP 分别运行固定 POI、地理编码和天气样本 | 工具/接口、字段层级、POI ID、经纬度、天气日期、空值形态与脱敏结果 |
| S1-M02 | 分别运行步行、驾车和公交样本 | distance、duration、polyline、路线 ID、公交方式细分及打车计费输入的可得性 |
| S1-M03 | 运行餐饮样本 | 营业时间存在率、缺失形态、分段/跨天表达及可确定解析边界 |
| S1-M04 | 对候选默认 Provider 执行同步串行连续调用 | 样本数、成功/失败数、耗时分布、超时表现和资源释放检查；样本规模必须在 ADR 中写明 |
| S1-M05 | 查验目标开发 Key 的官方控制台与官方规则 | 当日配额、限流、计费和查询日期；只记录规则与数值，不截图或抄录 Key |

live verification 必须通过 `backend/tests/live/verify_amap_provider.py` 显式执行并要求操作者选择 Provider；脚本启动前只报告 Key 是否已配置。实际命令、固定样本和循环次数在脚本 `--help` 与 ADR 中保持一致，避免文档和实现产生两套口径。

固定公开样本为北京：POI 分别检索“故宫博物院”“餐厅”“酒店”，地理编码使用“北京市东城区景山前街4号”，天气查询北京；三类路线统一使用天安门广场 `origin=(116.397477,39.908692)` 到颐和园 `destination=(116.275520,39.999340)` 的 GCJ-02 坐标。两种 Provider 各执行 8 次能力调用（3 POI + 1 地理编码 + 1 天气 + 3 路线）；候选默认 Provider 再将“景点 POI + 地理编码 + 天气 + 三类路线”6 个操作同步串行重复 5 轮，共 30 次。公开样本若因上游行政区调整失效，只能先更新本工作包并评审，不能在执行时临时换样本。

## 10. 量化退出指标

- [ ] S1-T01～S1-T08 全部通过，失败数为 0。
- [ ] S1-M01～S1-M05 全部形成脱敏记录；不适用项必须说明原因并判断是否阻断。
- [ ] POI ID、坐标、天气日期、路线 distance/duration/polyline 六类核心字段中，默认 Provider 均可稳定取得；任何必需字段缺失必须阻断对应后续 Stage，不能以 LLM 或默认值补写。
- [ ] MCP 连续调用没有遗留不可回收进程/会话；若 MCP 不满足，则 ADR 必须选择 HTTP 或明确阻断。
- [ ] 两个 Provider 的固定能力样本均完成 8 次调用；候选默认 Provider 的串行稳定性样本完成 30 次调用，核心字段有效次数为 30/30，未释放进程/会话数为 0。任何上游技术失败均如实记录并阻断选择该 Provider，不以重跑覆盖失败样本。
- [ ] ADR 明确回答默认 Provider、能力范围、失败识别、串行稳定性、调用预算、配额约束和已知缺口。
- [ ] 默认回归不访问公网、不读取真实 Key；提交内容 secret 命中数为 0，新增未追踪 TODO 数为 0。
- [ ] `03` 中 Stage 1 的每条退出条件均映射到上述自动化或人工验收 ID。

## 11. 迁移与回滚

- 本 Stage 不接入默认业务流量，回滚只需恢复 Provider 验证相关新增文件、配置字段和 pytest 依赖，不涉及用户数据或 TripPlan schema。
- 若某 Provider 验证失败，保留其失败 fixture 和脱敏结论作为决策证据；不得删除证据后宣称其可用，也不得在运行期自动回退到它。
- 回滚后旧 `/trip/plan` 仍使用原有链路；执行前端构建和 Stage 1 引入前仍适用的最小后端导入检查，结果记录在 `rollback-check.md`。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|------|------|----------|
| Provider ADR | `doc/opt_strategy/decisions/amap_provider_validation.md` | 能力矩阵、样本规模、字段完整性、错误形态、性能、配额来源、默认选择和风险 |
| 自动化摘要 | `doc/opt_strategy/evidence/stage_1/test-summary.md` | 提交标识、三条命令、通过/失败数和离线声明 |
| 人工验证 | `doc/opt_strategy/evidence/stage_1/manual-check.md` | S1-M01～M05 的期望、实际、结论和脱敏说明 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_1/rollback-check.md` | 回滚边界、命令、结果和数据影响 |

## 13. Ready/Done 复核

当前设计复核结论：**R2～R10 通过首轮跨 Stage 审计；R1 无前置依赖，在语义上满足**。但设计尚未冻结且未获得代码实施授权，Stage 1 仍不得开工；开工前须确认本工作包未变化，完成后再以真实测试/live 证据判定 Done。
