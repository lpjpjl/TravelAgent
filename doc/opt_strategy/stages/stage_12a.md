# Stage 12A：BudgetCalculator

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 12A |
| 目标 | 以确定性代码从本次真实事实、路线和住宿记录生成逐项费用、固定假设与“已知费用合计”，绝不让 Agent 或缺失数据补写价格 |
| 对应优化项 | P0-7 行程深度关联化（预算计算部分） |
| 前置依赖 | Stage 11 |
| 后续阻断 | 未通过时不得开始 Stage 12B、13A～13D、14、16 |
| 默认流量影响 | 无；只生成内部预算结果，legacy/shadow 普通请求与 `new` reserved 行为不变 |

## 2. 当前代码基线与差异

- legacy `Budget` 由 LLM/解析 fallback 填写整数总额，缺失价格通常变为 0，不能区分真实免费、未知价格与估算规则。
- Stage 9A/9B 已提供连续 lodging、真实餐饮 POI 与完整活动；Stage 10A/10B 提供实际路线方式/距离/耗时；Stage 11 提供日时间轴，但当前没有唯一预算责任者。
- 当前无 Decimal、计价单位、来源依据、人数/房间/晚数假设、unavailable 项或 partial 住宿规则；前端也尚未展示新模型。
- 打车与自驾不存在可审查的城市规则，不能从距离或 LLM 文本直接发明金额。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/budget.py` | 固化 Decimal 金额、费用项、计价单位、依据、quality、假设、missing items 与 known total 不变量 | 浮点/负金额、错误小计和 unavailable 入总计均拒绝 |
| 新增 | `backend/app/services/budget_calculator.py` | 纯函数计算门票、住宿、餐饮、transit、taxi/self_drive 与汇总 | 不导入 Agent、API、Provider 或前端 |
| 新增 | `backend/app/services/pricing_rules.py` | 读取并校验本地城市 taxi/self_drive 显式规则，按规则估算 | 缺规则返回 unavailable，不提供城市默认价 |
| 新增 | `backend/app/config/taxi_pricing_rules.json` | 仅存脱敏、版本化的本地开发计价规则样本 | 每条规则含城市标识、币种、版本和完整公式字段 |
| 修改 | `backend/app/config.py` | 增加规则文件 Settings 读取与路径校验 | 缺失/非法文件不泄漏路径内容；只使相关估算 unavailable |
| 修改 | `backend/.env.example` | 增加规则文件占位配置说明 | 不含真实 Key 或城市商业数据 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 在 Stage 11 后调用 BudgetCalculator 并保存内部预算 | 不据预算修改活动、路线或 Timeline |
| 新增 | `backend/tests/unit/services/test_pricing_rules.py` | 覆盖规则文件、边界、缺规则和公式 | 不读取正式环境文件 |
| 新增 | `backend/tests/unit/services/test_budget_calculator.py` | 覆盖费用项、单位、assumption、unavailable 与恒等式 | 固定 Decimal/时钟，离线执行 |
| 修改 | `backend/tests/unit/models/test_budget_trip_error.py` | 覆盖新预算 schema 与 JSON 十进制定点 | 金额精度与 quality 严格校验 |
| 新增 | `backend/tests/integration/test_trip_coordinator_budget.py` | 覆盖 Stage 11→12A、上游短路与下游边界 | 不生成 TripPlan 或调用 LLM |

本 Stage 不修改路线/时间轴算法、餐厅/酒店选择、Validator、最终 API/前端、legacy Agent 或缓存。Stage 12B 校验预算输入与完整日语义，Stage 13A 才组装正式 TripPlan。

## 4. 输入、输出与错误契约

### 4.1 输入、假设与可用范围

`BudgetCalculator.calculate()` 接收同一 request_id 的 ItineraryDraft、lodging records、真实 POI 事实、RouteSegments 和 Stage 11 TimelineCalculationResult：

- 只接受每个活动/路线/酒店 ID 可回填到同一请求来源事实的输入；未知 ID、重复费用对象、负 route distance、非 Decimal 来源价格或不连续 lodging 为 `invalid_budget_input` 技术校验错误；
- `scheduled` 与 `requires_repair` timeline 都可计算内部费用，以便后续修复重算；transit refresh pending/技术错误及上游 short-circuit 不调用 Calculator；
- 固定假设为 2 名成人、1 间房。完整 N 日 `lodging_count=N`；partial Day1..DayK 且 K<N 为 `K+1`，取 `lodging[0..K]`；单日完整为 1 晚。Calculator 不自行构造 partial；
- 所有金额使用 `Decimal`、币种 `CNY`、两位小数十进制字符串；`known_cost_total` 只累加 quality 为 sourced 或 estimated 的可用项，绝不称“总预算”。

### 4.2 费用项规则

| 类别 | 数量/单位 | 可用条件 | 缺失语义 |
|---|---|---|---|
| attraction ticket | 每张 × 2 人 | 来源明确成人单价 | `unavailable/source_price_missing` |
| lodging | 每晚 × 1 房 × lodging_count | 每个使用 lodging 有来源每晚价 | 对应晚 unavailable，不能以其他酒店价代替 |
| gourmet/convenience meal | 每餐 × 2 人 | 具体餐饮 POI 有来源人均/单人价格及单位 | unavailable；不得以名称、类别或 LLM 推价 |
| lodging_side breakfast | 仅在酒店事实明确含早或给出价格 | 含早计入住宿依据；明确每人价格才单列餐饮 ×2 | unavailable，不默认双人早餐费 |
| transit | 每段 × 2 人 | 来源明确本段单人票价 | unavailable，不按路线距离估票价 |
| taxi | 每段 × 1 车 | driving 路线距离/耗时 + 城市 taxi 规则 | 无规则/字段缺失为 unavailable |
| self_drive | 每段 × 1 车 | 明确城市里程与停车规则均存在 | 规则/事实任一缺失为 unavailable |

- taxi 规则固定字段为 `base_fare`、`base_distance_meters`、`per_km_after_base`、`per_minute`、`currency`、`rule_version`；费用为 base fare + 超出基准距离按向上取整公里计费 + duration 按向上取整分钟计费。规则字段、距离/时长或城市匹配缺失即 unavailable。
- self_drive 规则必须同时给出 `per_km` 与停车 `per_day`；缺任一字段不估算。taxi 路线仍标识为 taxi，不得误用 self_drive 规则。
- 每项保留 category、source/estimated quality、unit_price、quantity、unit、subtotal、currency、basis、source IDs 或 rule_version；含早已包含在住宿价格时不得重复添加早餐费用。

### 4.3 输出与错误

`BudgetCalculationResult` 至少包含：request_id、BudgetAssumption、按稳定类别/日期/ID排序的 items、known_cost_total、unavailable_items 与 `estimated_fields`：

- `subtotal = unit_price × quantity` 必须精确相等；每个可用项计入一次，unavailable 的 unit_price/subtotal 均为 null 且不计入 total；
- 事实价格为 `sourced`，规则估价为 `estimated`，汇总为 `computed`；禁止把规则金额标为 sourced；
- 合法缺价/缺规则是可展示 unavailable，不是技术错误。规则文件格式漂移、重复城市规则、非法 Decimal 或输入引用错误是稳定技术/校验错误；
- 日志仅记录 request_id、stage=`budget`、类别数量、available/unavailable 计数、known total、rule version 和耗时；不得记录规则原文、完整地址、自由文本或 Key。

## 5. 配置与功能开关

| Settings 字段 | 环境变量 | 默认值 | 缺失/非法行为 |
|---|---|---|---|
| `trip_taxi_pricing_rules_path` | `TRIP_TAXI_PRICING_RULES_PATH` | `config/taxi_pricing_rules.json` | 文件缺失或当前城市无规则时 taxi/self_drive 项 unavailable；JSON/schema 非法返回稳定配置错误 |

- 不新增流量开关；路线、餐饮、酒店与门票计算不依赖该文件。
- 路径只允许项目配置目录内的常规文件，启动日志仅记录“规则文件已加载/不可用”与版本摘要，不打印内容或绝对路径。
- `TRIP_PLANNER_MODE` 语义不变；new 在 Stage 16 前仍不对普通流量开放。

## 6. 实施步骤

1. 定义费用项/假设/result 严格模型，锁定 Decimal、unit、quality、unavailable 与总计恒等式。
2. 定义本地规则文件 schema 与 Settings 读取边界，验证 city、版本、币种、taxi/self_drive 公式字段。
3. 实现 ticket、lodging、餐饮、住宿侧早餐与 transit 的来源价格计算，不允许默认 0 或关键词估价。
4. 实现 taxi/self_drive 规则估算及缺规则分流；所有规则金额标记 estimated。
5. 实现 deterministic 排序、known total、missing items 与估算依据，接入 Coordinator 的 Stage 11 后内部路径。
6. 执行定向、Stage 1～12A 全量回归与前端构建，归档预算、规则和回滚证据。

## 7. 明确不做

- 不以预算约束或优化路线/活动，不查询实时价格、票务、打车平台或停车场。
- 不校验时间轴可执行性、营业时间、accepted constraints 或 partial 前缀；属于 Stage 12B/13D。
- 不输出 TripPlan/API/UI，不把 unavailable 隐藏为 0；前端展示属于 Stage 14。
- 不引入缓存、数据库、多币种、动态汇率、优惠券、价格敏感路线或 LLM 费用推理。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S12A-T01 | unit | 1/3 日 complete、Day1/Day1..2 partial | lodging_count 分别 1/3/2/3，使用 lodging 集合正确 |
| S12A-T02 | unit | 2 人门票/餐饮/transit 与 1 车 taxi | quantity/unit/subtotal 正确 |
| S12A-T03 | unit | 每类来源价格缺失 | unavailable 且不进入 known total，不回退 0 |
| S12A-T04 | unit | lodging_side 含早/明确早餐价/未知 | 分别仅住宿依据、单列 ×2、unavailable；无重复计费 |
| S12A-T05 | unit | taxi base/边界/公里分钟向上取整 | 固定规则公式与 Decimal 两位小数正确 |
| S12A-T06 | unit | taxi 无城市规则/字段缺失 | unavailable，不使用邻近城市或默认价 |
| S12A-T07 | unit | self_drive 完整/缺停车或里程规则 | 前者 estimated，后者 unavailable |
| S12A-T08 | unit | 混合/固定路线方式 | transit 按人、taxi 按车、self_drive 不混淆 |
| S12A-T09 | unit | known total 恒等式与排序 | 仅可用项求和；类别/日期/ID 稳定 |
| S12A-T10 | unit | 浮点、负数、错误小计/币种 | schema 拒绝 |
| S12A-T11 | unit | 规则文件合法/重复城市/非法 JSON | 合法读取；后两者稳定配置错误 |
| S12A-T12 | contract | Budget JSON | Decimal 字符串、quality、basis、unavailable null 形状固定 |
| S12A-T13 | integration | Stage 11→12A happy path | Coordinator 保存预算，不改 timeline/routes |
| S12A-T14 | integration | timeline requires_repair | 仍可计算内部预算，不构造 TripPlan |
| S12A-T15 | integration | 上游刷新/技术错误 | Calculator 调用数 0，状态原样短路 |
| S12A-T16 | regression | legacy/shadow/new 与上游 | 普通旧流量调用数 0；Stage 1～11/前端不回退 |

```text
python -m pytest backend/tests/unit/models/test_budget_trip_error.py backend/tests/unit/services/test_pricing_rules.py backend/tests/unit/services/test_budget_calculator.py backend/tests/integration/test_trip_coordinator_budget.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

三条命令期望退出码均为 `0`，不得访问公网、真实高德或真实 LLM。Stage 12A 必测用例数为 16。

## 9. 人工验收

不适用：本 Stage 不接入页面或 API。规则文件、计价、missing item 和总计由固定 fixture 自动覆盖；Stage 14 再人工核对预算呈现与“已知费用合计”文案。

## 10. 量化退出指标

- [ ] S12A-T01～S12A-T16 全部通过，失败数为 0，三条命令退出码为 0。
- [ ] 可用项 subtotal 与 unit_price×quantity 不一致数为 0；known total 包含 unavailable 项数为 0。
- [ ] complete/partial 住宿晚数与实际 lodging 集合不一致数为 0；单日末日晚餐后新增住宿数为 0。
- [ ] ticket/meal/transit 按人、taxi/self_drive 按车、lodging 按房晚的单位错误数为 0。
- [ ] 缺来源价格、规则或规则字段时生成默认价格/0 元/LLM 价格的次数为 0。
- [ ] 规则估价标为 sourced 或来源事实标为 estimated 的次数为 0。
- [ ] 默认测试外部调用数为 0，敏感规则/Key/地址/自由文本日志命中数为 0；Stage 1～11 回归和前端构建通过。

## 11. 迁移与回滚

- Stage 12A 仅新增内部预算和本地规则配置，不迁移用户数据、默认 API 或前端。
- 回滚先移除 Coordinator 的预算调用，再移除 Calculator/规则加载/模型与测试；保留 Timeline 和真实路线。规则文件可保留为无引用配置，不得作为 fallback 事实来源。
- Stage 12B 以后依赖时，按 13D→12B→12A 逆序回滚；回滚后执行 Stage 1～11 测试、模式回归和前端构建。
- 回滚不得恢复 legacy 的整数虚构预算或用 0 填充 unavailable。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_12a/test-summary.md` | 提交标识、三条命令、16 用例、离线声明和失败数 |
| 费用矩阵 | `doc/opt_strategy/evidence/stage_12a/budget-item-matrix.md` | 类别、来源/规则、单位、quantity、quality 与测试 ID |
| 住宿矩阵 | `doc/opt_strategy/evidence/stage_12a/lodging-count-matrix.md` | complete/partial/单日 lodging 集合、晚数与测试 ID |
| 规则矩阵 | `doc/opt_strategy/evidence/stage_12a/pricing-rule-matrix.md` | 城市、版本、公式边界、缺失分流与测试 ID |
| 回滚验证 | `doc/opt_strategy/evidence/stage_12a/rollback-check.md` | 调用撤销、配置影响、命令和结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批及 Stage 9A～16 跨 Stage 设计审计；执行 Ready 待 R1**。R1 必须等待 Stage 11 实际 Done，并在开工前复核 Timeline、路线方式、lodging 计数、来源价格字段、规则文件与 Stage 12B/13A 消费边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
