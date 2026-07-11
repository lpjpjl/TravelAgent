# 旅行 Agent Stage 代码实施规格

> 上游文档：`02_opt_plan.md`、`03_implement_plan.md`
>
> 文档状态：全局规范已建立，Stage 1～5 已拆分、细化并完成首轮跨 Stage 设计审计；后续按 Stage 分批补充并评审。未达到 04 “可开工”条件的 Stage，不应直接进入编码。

---

## 一、文档定位

### 1.1 三份文档的职责

| 文档 | 回答的问题 | 主要内容 |
|------|------------|----------|
| `02_opt_plan.md` | 为什么优化、必须满足什么 | 优化目标、产品规则、业务约束和优先级 |
| `03_implement_plan.md` | 按什么顺序实施、每阶段交付什么 | 目标架构、Stage 编排、依赖、范围、Gate 和阶段退出条件 |
| `04_stage_implementation_spec.md` + `stages/` | 具体改哪里、怎么实现、怎么验证 | 总纲承载全局规则与索引；Stage 分册承载文件迁移、接口契约、实施步骤、测试、证据和回滚 |

02 的产品语义原则上冻结。03 不重复承载函数级设计。本文不得改变 02 的业务约束、03 的 Stage 范围或依赖；发现冲突时停止细化并回到上游文档评审，不能在本文中静默改写。

### 1.2 约束优先级

同一问题存在多处描述时，按以下顺序处理：

1. 产品语义、业务约束和优化优先级以 02 为准；
2. Stage 顺序、依赖、交付边界和 Gate 以 03 为准；
3. 文件、函数、接口、测试、配置和回滚细节以本文为准；
4. 代码与本文不一致时，不默认以现状代码覆盖设计，应在对应 Stage 的基线差异中显式记录并评审。

### 1.3 本文维护规则

- `04_stage_implementation_spec.md` 是总纲，`stages/stage_<编号>.md` 是 Stage 分册；两者共同构成 04 的唯一权威规格，不因拆分形成新的文档层级。
- 全局规则、追踪矩阵和 Stage 索引只在总纲维护；工作包细节只在对应分册维护，禁止双份复制后分别演进。
- 每个 Stage 开工前补齐一个独立实施工作包；不得以其他 Stage 的工作包代替。
- 实现中发生文件名微调，可以更新本文；职责、schema 或 Stage 边界变化必须先评审上游影响。
- 所有“通过”“稳定”“显著”“主流”等定性用语必须转换为命令、样例、计数、阈值或检查清单。
- 外部服务能力、配额和兼容性等动态事实必须附实测日期与证据位置，不把未经验证的数字写成固定结论。
- 真实 Key、完整 Prompt、未脱敏上游响应、plan token 和用户自由文本不得进入文档、fixture 或验收记录。

---

## 二、Stage 可开工定义（Definition of Ready）

一个 Stage 只有同时满足以下条件才可进入编码；任一项为“否”即保持 `Not Ready`：

| 编号 | 必备条件 | 可判定证据 |
|------|----------|------------|
| R1 | 03 中列出的全部前置 Stage 和 Gate 已通过 | 对应验收记录链接或提交标识 |
| R2 | 当前代码基线及与目标的差异已列出 | 现有文件、入口和待迁移职责清单 |
| R3 | 修改、新增、删除文件范围已明确 | 文件变更表不存在“等”“相关文件” |
| R4 | 输入、输出、错误和状态契约已明确 | 字段表或最小 JSON 示例，可覆盖成功与失败 |
| R5 | 配置和功能开关已明确 | Settings 字段、默认值、作用域、移除阶段 |
| R6 | 实施步骤及明确不做事项已明确 | 有序步骤，每步可映射到文件和测试 |
| R7 | 自动化测试范围与命令已明确 | 测试文件、用例表、可复制执行命令和期望退出码 |
| R8 | 需要人工验收的场景已明确 | 环境、步骤、预期结果和记录位置 |
| R9 | 退出指标全部可判断 | 每项为布尔条件、明确阈值或精确计数 |
| R10 | 迁移与回滚可执行 | 开关/入口切换步骤、回滚步骤和回滚后验证命令 |

“已讨论”“开发者自行判断”“测试正常”均不能作为 Ready 证据。

设计冻结与执行 Ready 使用同一张 R1～R10 清单，但判定时点不同：设计阶段必须完成 R2～R10 的规格复核，并标记为“设计完整，执行 Ready 待 R1”；R1 只有在前置 Stage 实际实施并通过后才能成立。设计冻结不要求尚未实施的后续 Stage 虚构 R1 证据，任何 Stage 真正进入编码前仍必须补齐 R1，并再次确认 R2～R10 未因前序实现变化而失效。

---

## 三、Stage 完成交付定义（Definition of Done）

03 的通用完成定义继续有效。本文进一步要求每个 Stage 提交以下证据：

1. 文件变更与工作包声明一致；超出范围的变更有评审记录。
2. 工作包列出的自动化命令退出码全部为 `0`，并记录执行日期、环境和摘要。
3. 所有必测用例均通过；禁止用降低断言、删除 fixture 或扩大容错代替修复。
4. 人工验收项逐条记录实际结果；不适用项必须说明原因。
5. API/schema 变化已同步后端模型、前端类型、示例和接口说明。
6. 新增配置均通过统一 Settings 入口读取，示例环境文件只出现占位值。
7. 日志和错误响应中不包含敏感数据，且满足 `request_id` 传播要求。
8. 回滚步骤至少完成一次可重复验证；纯新增且未接入默认链路的能力可用关闭开关验证。
9. 上一 Stage 的验收命令仍通过；若命令尚未建立，执行工作包指定的最小回归集合。
10. 退出条件证据已归档，Stage 状态才可由 `In Progress` 改为 `Done`。

建议将记录存放在 `doc/opt_strategy/evidence/stage_<编号>/`。二进制产物、含敏感信息的原始响应和体积较大的运行日志不提交仓库，只提交脱敏摘要及其受控位置。

---

## 四、统一量化口径

后续工作包必须使用以下表达方式，具体数值在对应 Stage 中确定：

| 模糊表达 | 必须替换为 |
|----------|------------|
| “测试通过” | 完整命令、期望退出码、必测用例列表 |
| “结果稳定” | 固定输入重复次数、允许差异字段、通过阈值 |
| “性能可接受” | 数据规模、采样次数、统计口径和耗时上限 |
| “显著减少请求” | 优化前基线、优化后上限及浏览器统计方式 |
| “有限重试” | 最大总次数、同类最大次数、停止条件 |
| “主流 PC 宽度” | 浏览器版本、视口宽高和缩放比例矩阵 |
| “部分失败可用” | 成功/失败样例数量、最终状态和页面/API 预期 |
| “人工验收通过” | 编号步骤、预期、实际结果、截图或记录路径 |
| “可回滚” | 回滚命令或开关、数据影响、验证命令和预期结果 |

量化指标服务于可重复判断，不应凭空设定外部 API 的成功率、配额或延迟承诺。相关阈值应先通过 Stage 1 等技术验证获得基线。

---

## 五、统一 Stage 实施工作包模板

后续每个 Stage 使用以下模板。标记为“不适用”的栏目也必须保留并说明原因。

### Stage X：名称

#### 1. 工作包元数据

| 字段 | 内容 |
|------|------|
| 对应 03 章节 | `03_implement_plan.md#...` |
| 对应优化点 | P0/P1/P2/P3 编号 |
| 状态 | `Not Ready / Ready / In Progress / Done` |
| 前置 Stage | 明确编号 |
| 前置 Gate | Gate 名称及证据位置 |
| 预计变更边界 | 后端/前端/配置/文档/依赖 |

#### 2. 当前代码基线与差异

- 当前入口：精确到文件、类或函数。
- 当前行为：用最小输入输出描述，不复制大段代码。
- 与目标差异：逐项对应 03 的本 Stage 范围。
- 复用内容：明确保留的代码和职责。
- 待迁移/废弃内容：说明停止扩展和最终删除的 Stage。

#### 3. 文件变更清单

| 操作 | 文件 | 主要符号/职责 | 完成判据 |
|------|------|---------------|----------|
| 修改/新增/删除 | 精确相对路径 | 类、函数、组件或配置项 | 可检查结果 |

不得使用“相关文件”“视情况新增”等开放描述。确实无法预先确定时，本 Stage 保持 `Not Ready`。

#### 4. 输入、输出与错误契约

- 输入模型：字段、类型、必填性、取值范围和校验失败行为。
- 输出模型：字段、类型、状态组合和数据来源要求。
- 错误模型：稳定错误码、HTTP 状态、是否可重试及前端行为。
- 兼容策略：旧/新 schema 共存范围、切换点和删除点。
- 示例：至少包含一个最小成功样例和本 Stage 涉及的关键失败样例。

#### 5. 配置与功能开关

| 名称 | Settings 字段 | 环境变量 | 默认值 | 生效范围 | 引入/移除 Stage |
|------|---------------|----------|--------|----------|-----------------|

说明缺失配置的启动行为、日志脱敏要求以及开关开启、关闭时分别走哪条入口。

#### 6. 实现步骤

按执行顺序列出。每一步必须同时引用文件变更项、契约项和对应测试，不写函数内部伪代码之外的无约束方案。

1. 第一步及可观察结果。
2. 第二步及可观察结果。
3. 接入、迁移和清理步骤。

#### 7. 明确不做

- 列出容易被误带入本 Stage 的相邻能力。
- 指明由哪个后续 Stage 承担；没有对应 Stage 时标记为范围缺口并暂停评审。

#### 8. 自动化测试

**测试文件**

| 测试文件 | 层级 | 覆盖对象 |
|----------|------|----------|
| 精确相对路径 | 单元/契约/API/组件 | 类、接口或状态机 |

**必测用例**

| ID | 输入/前置条件 | 预期输出 | 关键断言 |
|----|---------------|----------|----------|
| SX-T01 | 精确条件 | 精确状态 | 可自动判断的断言 |

**执行命令**

```text
从仓库根目录可复制执行的命令
```

记录期望退出码、是否访问外网、所需 fixture，以及该命令属于本 Stage 验收还是回归集合。

#### 9. 人工验收

| ID | 环境与步骤 | 预期结果 | 证据 |
|----|------------|----------|------|
| SX-M01 | 浏览器/视口/操作或外部实测过程 | 可见且可判断的结果 | 记录文件或截图名称 |

没有人工验收时明确写“不适用：全部行为已由自动化测试覆盖”。

#### 10. 量化退出指标

采用全真条件，不使用加权评分：

- [ ] 03 中本 Stage 的每条退出条件均映射到测试或人工验收 ID。
- [ ] 必测自动化用例通过数 = 总数，失败数 = 0。
- [ ] 必须人工验收项通过数 = 总数，阻断问题数 = 0。
- [ ] 新增未追踪 TODO 数 = 0；已接受 TODO 有负责人和目标 Stage。
- [ ] 本 Stage 指定的最小回归集合失败数 = 0。

具体 Stage 在此基础上补充业务数量、边界值、请求数、耗时或兼容性指标。

#### 11. 迁移与回滚

- 迁移前状态和数据影响。
- shadow/功能开关启用顺序。
- 默认入口切换步骤。
- 回滚触发条件。
- 回滚操作及是否涉及数据清理。
- 回滚后验证命令、期望退出码和预期行为。
- 临时兼容代码或开关的删除 Stage。

#### 12. 交付证据

| 证据 | 位置 | 必填内容 |
|------|------|----------|
| 自动化测试摘要 | `evidence/stage_x/...` | 日期、环境、命令、退出码、通过/失败数 |
| 人工验收记录 | `evidence/stage_x/...` | 用例 ID、实际结果、证据引用 |
| 技术决策记录 | ADR 路径或“不适用” | 选择、备选方案、理由和影响 |
| 回滚演练记录 | `evidence/stage_x/...` | 操作、验证结果、遗留问题 |

#### 13. Ready/Done 复核

- Ready 复核：R1～R10 全部满足，评审日期与结论明确。
- Done 复核：本文第三章及本工作包退出指标全部满足。
- 未关闭事项：只能列入有负责人、目标 Stage 和阻断级别的事项。

---

## 六、全局追踪矩阵

### 6.1 优化项到 Stage 的覆盖矩阵

本表将 02 的优化项映射到 03 的交付 Stage。`工作包状态` 表示本文是否已经补齐代码级实施规格，不表示功能是否完成。

| 优化项 | 03 Stage | 最终 Gate | 04 工作包状态 | 验收证据索引 |
|--------|----------|-----------|---------------|--------------|
| P0-1 AmapService 结构化解析 | 1、3、4 | A | Stage 1、3、4 已细化 | `evidence/stage_1/`、`evidence/stage_3/`、`evidence/stage_4/` |
| P0-2 标准化数据调用链 | 5 | B | Stage 5 已细化 | `evidence/stage_5/` |
| P0-3 多偏好完整检索 | 5、6 | B | Stage 5、6 已细化 | `evidence/stage_5/`、`evidence/stage_6/` |
| P0-4 结构化草案输出 | 2、5、7B、8、9B | B | Stage 2、5、7B、8、9B 已细化 | `evidence/stage_2/`、`evidence/stage_5/`、`evidence/stage_7b/`、`evidence/stage_8/`、`evidence/stage_9b/` |
| P0-5 餐饮候选检索与筛选 | 7A、7B、9A、9B | B | Stage 7A、7B、9A、9B 已细化 | `evidence/stage_7a/`、`evidence/stage_7b/`、`evidence/stage_9a/`、`evidence/stage_9b/` |
| P0-6 RouteService 基础版 | 10A、10B | B | Stage 10A、10B 已细化 | `evidence/stage_10a/`、`evidence/stage_10b/` |
| P0-7 行程深度关联化 | 7A～13D | B | Stage 7A、7B、8、9A、9B、11、12A、12B、13A～13D 已细化 | `evidence/stage_7a/`、`evidence/stage_7b/`、`evidence/stage_8/`、`evidence/stage_9a/`、`evidence/stage_9b/`、`evidence/stage_11/`、`evidence/stage_12a/`、`evidence/stage_12b/`、`evidence/stage_13a/`、`evidence/stage_13b/`、`evidence/stage_13c/`、`evidence/stage_13d/` |
| P0-8 可执行时间轴信息 | 11、14 | B | Stage 11、14 已细化 | `evidence/stage_11/`、`evidence/stage_14/` |
| P0-9 对话式行程迭代 | 31A～32 | E | 待补齐 | 对应 Stage 目录 |
| P1-1 分级 Fallback 与结果状态 | 16 | C | Stage 16 已细化 | `evidence/stage_16/` |
| P1-2 分级错误处理 | 15 | C | Stage 15 已细化 | `evidence/stage_15/` |
| P1-3 请求缓存与外部数据降级 | 17 | C | Stage 17 已细化 | `evidence/stage_17/` |
| P1-4 地图信息增强 | 20 | C | Stage 20 已细化，R2～R10 跨 Stage 审计通过 | `evidence/stage_20/` |
| P1-5 天气驱动决策 | 18 | C | Stage 18 已细化，R2～R10 跨 Stage 审计通过 | `evidence/stage_18/` |
| P1-6 地图 Key 风险治理 | 19 | C | Stage 19 已细化，R2～R10 跨 Stage 审计通过 | `evidence/stage_19/` |
| P2-1 当前行程导航与 PDF 历史 | 13E、26、27 | D | Stage 13E 已细化；26、27 待补齐 | `evidence/stage_13e/`、`evidence/stage_26/`、`evidence/stage_27/` |
| P2-2 SSE 流式进度 | 28 | D | 待补齐 | `evidence/stage_28/` |
| P2-3 导出代码重构 | 25 | D | Stage 25 已细化，R2～R10 独立审计通过 | `evidence/stage_25/` |
| P2-4 图片批量接口 | 23 | D | Stage 23 已细化，R2～R10 跨 Stage 审计通过 | `evidence/stage_23/` |
| P2-5 图片加载优化 | 24 | D | Stage 24 已细化，R2～R10 跨 Stage 审计通过 | `evidence/stage_24/` |
| P2-6 前端代理统一 | 21 | D | Stage 21 已细化，R2～R10 独立审计通过 | `evidence/stage_21/` |
| P2-7 天级独立折叠 | 22 | D | Stage 22 已细化，R2～R10 跨 Stage 审计通过 | `evidence/stage_22/` |
| P2-8 PC 显示稳定性 | 29 | D | 待补齐 | `evidence/stage_29/` |
| P2-9 行前须知 | 30 | D | 待补齐 | `evidence/stage_30/` |
| P3-1 统一配置管理 | 33 | E 后收口 | 待补齐 | `evidence/stage_33/` |

Future F-1～F-4 不进入 Stage 工作包、Gate 或完成率统计。若实现范围发生变化，必须先修改 02/03，不能只在本表添加。

### 6.2 Stage 到 Gate 的证据矩阵

| Gate | 关闭 Stage | 必须聚合的证据 | Gate 记录 |
|------|------------|----------------|-----------|
| A 真实数据链可用 | 1～4 | Provider 决策、脱敏响应 fixture、解析契约测试、异常形态测试 | `evidence/gate_a/summary.md` |
| B 核心规划闭环可用 | 5～13D，含 13E 的前端 schema 准备 | 约束、候选、两次草案、路线、时间、预算、校验、修复、partial/failed 测试 | `evidence/gate_b/summary.md` |
| C 开发环境可靠链路可用 | 14～20 | 错误与 fallback、缓存降级、天气重建、Key 隔离、真实地图路线 | `evidence/gate_c/summary.md` |
| D P2 体验闭环 | 21～30 | 代理、折叠、图片、PDF、历史、SSE、PC 矩阵、行前须知 | `evidence/gate_d/summary.md` |
| E 对话迭代可上线 | 31A～32 | token 完整性、intent、重算、冲突、原子替换与会话恢复 | `evidence/gate_e/summary.md` |

Gate 汇总只引用各 Stage 的证据，不复制测试日志。Gate 的所有条件按全真规则判断，不使用通过率评分。

### 6.3 当前代码到目标职责的迁移索引

这是全局导航，不代替各 Stage 的精确文件变更表。

| 当前基线 | 当前职责/问题 | 目标位置 | 主要迁移 Stage |
|----------|---------------|----------|----------------|
| `backend/app/models/schemas.py` | 请求、行程和展示模型集中 | `backend/app/models/source.py`、`constraint.py`、`candidate.py`、`itinerary.py`、`trip.py` | 2；后续按使用点增量扩展 |
| `backend/app/agents/trip_planner_agent.py` | 工具调用、规划、解析、虚构 fallback 混合 | `agents/`、`coordinators/trip_coordinator.py`、确定性 services | 5～16 |
| `backend/app/services/amap_service.py` | 高德能力入口尚不完整 | `services/amap_service.py` + `providers/` | 1、3、4、17 |
| `backend/app/api/routes/trip.py` | 同步业务直接挂在异步路由且错误统一为 500 | `api/routes/trip.py` + Coordinator 线程池调用 | 13A、15、16 |
| `frontend/src/types/index.ts` | 旧 TripPlan 类型集中 | 与 Stage 2/16 后端 schema 对齐的领域类型 | 2 先定义契约，13E/16 接入 |
| `frontend/src/services/api.ts` | 硬编码 base URL、仅 JSON 请求 | 统一 JSON/stream/multipart 请求入口 | 21、28 |
| `frontend/src/views/Result.vue` | 展示与本地交互集中 | 只读时间轴、地图、导出及对话组件 | 13E、14、20、22～32 |
| 当前无 `backend/tests/` | 无自动化后端测试基线 | 本文第七章测试结构 | 首次在 Stage 1 引入 |
| 当前无前端测试框架 | 仅 `vue-tsc + vite build` | 构建检查 + 记录化人工验收；必要时由具体 Stage 提案引入组件测试 | 暂不预装测试框架 |

---

## 七、全局工程与测试规范

### 7.1 当前基线与引入原则

- 当前后端 `requirements.txt` 没有 pytest，仓库也没有 pytest 配置或测试目录。
- 当前前端只有 `dev`、`build`、`preview` 脚本，没有组件或端到端测试框架。
- Stage 1 在其变更集中引入后端最小测试工具链；本文不提前修改依赖文件。
- 前端继续以严格类型检查、生产构建和记录化人工验收为默认。只有某 Stage 存在高价值、可确定性测试的前端状态逻辑时，才在该工作包中提出测试框架和独立评审，不为形式统一预装完整平台。

### 7.2 后端目标测试结构

采用统一根目录 `backend/tests/`：

```text
backend/tests/
  conftest.py
  fixtures/
    amap/                 # 脱敏后的 MCP/HTTP 响应与异常形态
    llm/                  # 最小合法/非法 JSON，不保存完整 Prompt
  unit/                   # 纯模型、解析器、Calculator、Validator、策略测试
  contract/               # Provider、Agent 结构化输出、schema 契约测试
  api/                    # FastAPI 路由、错误模型、SSE 和报告接口测试
  integration/            # fake Provider + fake LLM 的进程内链路，不访问公网
```

命名规则：测试文件使用 `test_<对象>.py`，用例使用 `test_<行为>__<条件>__<结果>`。各 Stage 工作包中的用例 ID 使用 `S<编号>-T<两位序号>`；例如 `S10A-T03`。

### 7.3 测试分层边界

| 层级 | 允许依赖 | 禁止事项 | 默认执行属性 |
|------|----------|----------|----------------|
| unit | 纯函数、模型、内存对象 | 网络、真实 LLM、真实 Key、时间不受控 | 每个相关 Stage 必跑 |
| contract | 脱敏 fixture、fake adapter | 把实时响应当唯一断言来源 | Provider/schema Stage 必跑 |
| api | FastAPI 测试客户端、临时目录 | 写正式报告目录、真实外部调用 | API 变更 Stage 必跑 |
| integration | fake Provider、fake LLM、固定时钟 | 公网和真实配额 | 调用链/修复 Stage 必跑 |
| live verification | 本地显式 Key 和受控请求 | 混入默认 pytest、写入敏感响应 | 仅技术验证 Stage 人工触发 |

所有默认自动化测试必须离线、可重复、不会消耗高德或 LLM 配额。需要实时服务的验证必须使用独立命令/标记，并将脱敏摘要作为证据。

### 7.4 fixture 与 fake 规范

- fixture 必须注明来源通道、采集日期、脱敏方式和覆盖场景；坐标、POI ID 等用于真实性契约的业务字段可以保留，Key、用户文本和请求签名必须删除。
- 原始响应只保留验证所需的最小字段，不整包提交未知敏感内容。
- `FakeAmapProvider` 按调用序列返回固定 DTO 或稳定异常，并记录方法、参数摘要和调用次数。
- `FakeLLM` 只返回工作包声明的 JSON 字符串或异常；禁止通过真实模型“碰巧成功”完成自动化验收。
- 时间轴、缓存和 token 测试使用可注入时钟；不得依赖 `sleep` 判断 TTL 或超时。
- PDF/API 文件测试使用 pytest 临时目录；不得读写正式 `TRIP_REPORT_DIR`。

### 7.5 标准命令口径

Stage 1 引入 pytest 后，工作包应从仓库根目录使用以下命令形态，并按实际测试文件收窄范围：

```text
python -m pytest backend/tests/<层级>/<测试文件> -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

- 第一条是 Stage 定向验收；第二条是后端最小回归；第三条同时承担 TypeScript 类型检查和生产构建。
- 每个命令期望退出码均为 `0`。工作包不得只写 `pytest`、`npm test` 等当前仓库不存在或依赖工作目录的模糊命令。
- live verification 命令在 Stage 1 工作包中单独定义，默认回归不得执行。
- 本轮不设置统一代码覆盖率百分比。每个确定性分支必须由用例表逐项覆盖，避免用总体覆盖率掩盖关键规则遗漏。

### 7.6 人工验收和证据规范

采用 `doc/opt_strategy/evidence/` 保存可审查的脱敏 Markdown 摘要。目标结构为：

```text
doc/opt_strategy/evidence/
  stage_<编号>/
    test-summary.md
    manual-check.md
    rollback-check.md
  gate_<字母>/
    summary.md
```

每份记录至少包含：提交标识、日期、环境、命令或步骤、期望结果、实际结果、通过/失败结论和未关闭问题。截图只在文字无法证明视觉结果时使用；不提交 PDF、完整日志、Key、token、Prompt 或未脱敏响应。

---

## 八、配置规范

### 8.1 单一读取边界

- 后端新增配置必须先进入 `backend/app/config.py` 的 `Settings`，业务模块不得新增直接 `os.getenv()`；Stage 33 负责清理当前遗留的重复读取。
- 后端环境变量使用大写 `UPPER_SNAKE_CASE`，Settings 字段使用对应小写 `lower_snake_case`。
- 浏览器可见配置仅使用 `VITE_` 前缀；后端 Key、plan token secret 等严禁进入前端环境和构建产物。
- `.env.example` 只写占位值、用途和是否必填，不写真实密钥。
- 配置首次被功能使用的 Stage 即接入 Settings，不推迟到 Stage 33。

### 8.2 已规划配置登记表

具体默认值和范围由首次使用它的 Stage 工作包定稿；Stage 1 技术验证前不臆造 TTL、配额或超时数字。

| 环境变量 | 类型/敏感性 | 首次使用 Stage | 缺失行为 |
|----------|-------------|----------------|----------|
| `AMAP_PROVIDER` | 枚举，非敏感 | 1 | 非法值启动失败；默认通道由 Stage 1 ADR 确定 |
| `AMAP_API_KEY` | secret | 1 | 使用 HTTP/MCP 高德能力时启动或首次调用快速失败，工作包定稿 |
| `VITE_AMAP_WEB_JS_KEY` | 浏览器公开 Key | 19 | 地图能力关闭或明确报配置错误，不回退后端 Key |
| `TRIP_CACHE_MAX_ENTRIES` | 正整数 | 17 | 非法值启动失败 |
| `TRIP_CACHE_TTL_POI_SECONDS` | 正整数 | 17 | 非法值启动失败 |
| `TRIP_CACHE_TTL_WEATHER_SECONDS` | 正整数 | 17 | 非法值启动失败 |
| `TRIP_CACHE_TTL_ROUTE_SECONDS` | 正整数 | 17 | 非法值启动失败 |
| `TRIP_TAXI_PRICING_RULES_PATH` | 项目内规则文件路径，非敏感 | 12A | 文件缺失/当前城市无规则时相关估算 unavailable；JSON/schema 非法返回稳定配置错误 |
| `TRIP_REPORT_DIR` | 持久化目录 | 26 | 报告能力不可用并返回明确配置错误，不使用系统临时目录替代 |
| `TRIP_REPORT_MAX_UPLOAD_MB` | 正数 | 26 | 非法值启动失败 |
| `PLAN_TOKEN_SECRET` | secret | 31A | 修订能力不可启用，禁止使用硬编码默认 secret |
| `TRIP_REPAIR_MAX_ATTEMPTS` | 非负整数 | 13B | 非法值启动失败 |
| `TRIP_REPAIR_MAX_SAME_TYPE_ATTEMPTS` | 非负整数且不大于总上限 | 13B | 非法组合启动失败 |

LLM、Unsplash 和当前已有应用配置继续沿用现有字段，首次相关 Stage 应记录基线差异；Stage 33 只做读取收口和校验清理，不延迟功能所需配置。

### 8.3 配置验收

每个新增配置至少覆盖：合法值、缺失值、非法类型/范围、日志脱敏四类检查。secret 只能验证“已配置/未配置”，不能出现在异常、日志、测试摘要或 API 响应中。

---

## 九、功能开关、迁移与回滚规范

### 9.1 开关登记表

首版统一使用以下语义名称；最终环境变量名在对应工作包确认后不得产生同义重复项。

| 建议环境变量 | 控制能力 | 引入 Stage | 默认开启 Stage | 计划移除/复核 |
|--------------|----------|------------|----------------|---------------|
| `TRIP_PLANNER_MODE=legacy|shadow|new` | `/trip/plan` 规划链路选择 | 5 | 16 切为 `new` | 新链路稳定后单独清理 legacy，再收敛为无开关 |
| `TRIP_SSE_ENABLED` | SSE 进度入口/UI | 28 | 28 验收后 | Stage 33 复核是否保留 |
| `TRIP_REPORT_HISTORY_ENABLED` | PDF 上传、列表和历史 UI | 26/27 | 27 验收后 | Stage 33 复核是否保留 |
| `TRIP_REVISION_ENABLED` | 修订 API 与对话 UI | 31A/32 | Gate E 后 | Stage 33 复核是否保留 |

不为 Stage 1～4 增加用户流量开关，它们通过独立测试/live verification 入口验证。地图、图片和普通页面局部优化默认随对应 Stage 交付，除非工作包能说明独立关闭后的完整用户行为。

### 9.2 `TRIP_PLANNER_MODE` 精确语义

| 模式 | 普通 `/trip/plan` 响应来源 | 新链路执行 | 失败影响 |
|------|----------------------------|------------|----------|
| `legacy` | 旧链路 | 不执行 | 旧链路必须在 Stage 16 前移除虚构 fallback |
| `shadow` | 旧链路 | 仅显式测试入口或明确启用的开发验证中执行；普通用户请求不得无条件双跑外部调用 | 新链路结果只记录脱敏差异，不改变响应 |
| `new` | TripCoordinator 新链路 | 执行 | 按分级错误及 complete/partial/failed 语义返回，不调用虚构 fallback |

模式切换必须由 Settings 在启动时校验，日志记录模式但不记录 Key、Prompt 或完整请求。shadow 对比必须定义可比较字段和允许差异，不能用 JSON 全文相等作为规划质量结论。

阶段语义：Stage 5～15 期间 `new` 是保留值，dispatcher 使用统一错误模型返回 503 + `planner_mode_not_ready`；只有 Stage 16 完成默认链路切换后，上表 `new` 行的正式语义才生效。

### 9.3 独立能力开关语义

- 关闭 SSE：保留普通同步规划入口，前端不显示假进度。
- 关闭报告历史：不得影响浏览器本地 PDF 下载；隐藏上传、列表和历史入口，不删除已有 PDF。
- 关闭修订：当前 TripPlan 仍可只读查看；隐藏对话入口，修订 API 返回稳定的能力未启用错误。
- 前端与后端必须使用同一能力状态，禁止只隐藏 UI 而后端无保护，或后端关闭但 UI 仍可提交。

### 9.4 开关生命周期与回滚证据

每个工作包必须填写：引入值、默认值、切换前置 Gate、回滚目标值、数据影响和删除 Stage。回滚记录至少证明：

1. 切换后进程使用预期入口；
2. 原有持久化 PDF 不被删除或覆盖；
3. 当前结构化行程不会被旧/迟到响应覆盖；
4. 回滚路径不存在虚构事实兜底；
5. 回滚后的定向验收与最小回归命令退出码均为 `0`。

功能开关是迁移工具，不是永久维护两套 schema 或两套业务逻辑的理由。临时开关没有明确移除条件时，对应 Stage 不满足 Ready。

---

## 十、已确认的全局决策

以下决策已经用户确认，后续 Stage 工作包直接遵循，不再重复评审目录选择：

1. 以 `backend/tests/` 作为统一后端测试根目录，并在 Stage 1 引入最小 pytest 工具链。
2. 将脱敏 Markdown 验收摘要提交到 `doc/opt_strategy/evidence/`；不提交大日志、二进制产物或敏感原始响应。

确认状态：已确认。

---

## 十一、Stage 工作包索引

Stage 完整工作包拆分到 `stages/`，与本文共同构成 04 权威实施规格。本文只维护全局规则、追踪矩阵与索引；字段、测试和回滚细节只在对应 Stage 文件维护，不在总纲重复。

| Stage | 工作包 | 规格状态 | Ready 状态 |
|-------|--------|----------|------------|
| 1 | [高德 MCP 与官方 HTTP Provider 技术验证](stages/stage_01.md) | 已细化，R2～R10 设计复核通过 | R1 无前置依赖；执行授权待设计冻结 |
| 2 | [核心领域契约](stages/stage_02.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 1 Done 后补 R1 |
| 3 | [AmapService POI 与地理编码结构化解析](stages/stage_03.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 1、2 Done 后补 R1 |
| 4 | [AmapService 天气与路线结构化解析](stages/stage_04.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 1～3 Done 后补 R1 |
| 5 | [额外要求解析与标准化数据调用链 shadow 接入](stages/stage_05.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 1～4 Done 后补 R1 |
| 6 | [多偏好完整检索](stages/stage_06.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 5 Done 后补 R1 |
| 7A | [主要点位候选池](stages/stage_07a.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 5、6 Done 后补 R1 |
| 7B | [DestinationResearchAgent 主要点位筛选](stages/stage_07b.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 5、7A Done 后补 R1 |
| 8 | [PrimaryItineraryDraft](stages/stage_08.md) | 已细化，R2～R10 设计复核通过 | 待 Stage 2、5、7B Done 后补 R1 |
| 9A～16 | 对应 Stage 分册 | 已细化，R2～R10 全范围一致性审计通过 | 各 Stage 仍须待实际前置 Done 后补 R1；16 另待 Gate B 实际证据 |
| 17 | [进程内请求缓存与外部数据降级](stages/stage_17.md) | 已细化，R2～R10 独立设计复核通过 | 待 Stage 1、16 Done 后补 R1 |
| 18 | [天气驱动两次草案规划](stages/stage_18.md) | 已细化，R2～R10 跨 Stage 审计通过 | 待 Stage 4、13D、16、17 Done 后补 R1 |
| 19 | [本地地图 Key 风险治理](stages/stage_19.md) | 已细化，R2～R10 跨 Stage 审计通过 | 待 Stage 1 Done 后补 R1 |
| 20 | [地图真实路线与时间轴联动](stages/stage_20.md) | 已细化，R2～R10 跨 Stage 审计通过 | 待 Stage 10B、14、16、17、19 Done 后补 R1 |
| 21 | [本地开发前端代理统一](stages/stage_21.md) | 已细化，R2～R10 独立审计通过 | 待 Stage 19 Done 后补 R1 |
| 22 | [多天独立折叠与全部展开](stages/stage_22.md) | 已细化，R2～R10 跨 Stage 审计通过 | 待 Stage 14 Done 后补 R1 |
| 23 | [图片批量接口](stages/stage_23.md) | 已细化，R2～R10 跨 Stage 审计通过 | 待 Stage 13A、21 Done 后补 R1 |
| 24 | [图片加载优化](stages/stage_24.md) | 已细化，R2～R10 跨 Stage 审计通过 | 待 Stage 22、23 Done 后补 R1 |
| 25 | [导出重构、自动全展开与 PDF 下载](stages/stage_25.md) | 已细化，R2～R10 独立审计通过 | 待 Stage 20、22、24 Done 后补 R1 |
| 26～33 | 按 03 顺序逐批建立对应分册 | 待补齐 | Not Ready |

### 11.1 Stage 1～5 首轮跨阶段设计审计

审计日期：2026-07-06。审计范围仅为 04 总纲与 Stage 1～5 分册，不代表代码、live verification 或 Gate 已通过。

| 审计维度 | 结论 | 关键收口 |
|----------|------|----------|
| 依赖与实施顺序 | 通过 | Stage 1→2→3→4→5 的事实、模型、解析、编排依赖闭合；R1 改为实施时逐 Stage 解锁 |
| 文件职责与增量修改 | 通过 | Provider、领域模型、Provider 适配器、共享 parser、AmapService、Coordinator/dispatcher 各有唯一职责；重复文件均明确为后续 Stage 增量修改 |
| 模型与来源语义 | 通过 | DataQuality 对齐 02 的五类；SourceMetadata 按 POI/天气/路线分别约束标识；generated route ID 不改变路线 sourced 事实 |
| 调用上下文 | 通过 | request_id 从 API/显式验证入口贯穿 Coordinator、Agent、AmapService、Provider、日志和错误，且不进入事实 ID、查询条件或缓存键 |
| Provider/解析边界 | 通过 | ProviderRawResult 保留运行期完整 payload；仓库 fixture 单独最小化脱敏；MCP/HTTP 适配器提取后进入共享领域 parser |
| 空结果与错误 | 通过 | 合法空结果、unavailable、部分坏条目、AmapParseError、ProviderError 和可信业务 failed 不互相冒充 |
| shadow 与迁移 | 通过 | legacy/shadow 普通请求零双跑；new 在 Stage 16 前固定返回 503 planner_mode_not_ready；无不完整 TripPlan 外泄 |
| 测试与量化 | 通过 | Stage 1～5 分别固定 8/17/15/17/22 个自动化用例；Stage 1 live 样本、调用数和串行轮次已量化 |
| 回滚与敏感信息 | 通过 | 每个 Stage 可按依赖逆序回滚；Key、完整 Prompt、自由文本、完整 payload 不进入日志或证据 |

审计结论：Stage 1～5 的 R2～R10 设计材料通过首轮一致性复核。Stage 1 因无前置 Stage，R1 在语义上满足，但在设计冻结且用户明确授权前仍不得编码；Stage 2～5 的 R1 必须等待各自前置 Stage 实际 Done 后逐项补证。

### 11.2 Stage 6～8 第二次跨阶段设计审计

审计日期：2026-07-09。审计范围为 04 总纲、Stage 6、7A、7B、8 分册，并对照 02/03 中 P0-3、P0-4、P0-5、P0-7、P1-2 的业务边界；不代表代码、测试、Gate 或设计冻结已完成。

| 审计维度 | 结论 | 关键收口 |
|----------|------|----------|
| 依赖与实施顺序 | 通过 | Stage 6→7A→7B→8 的输入输出链闭合；Stage 9A 在本批审计通过前保持阻断，R1 仍待前置 Stage 实际 Done 后逐项补证 |
| ID、required 与约束传递 | 通过 | `PreferenceSearchResult` 独立保留 required POI；`PrimaryCandidatePool` 合并 required 与 accepted `constraint_ids`；`PrimaryCandidateSelection` 强制 required 全保留；`PrimaryItineraryDraft` 要求 selection ID 恰好安排一次且 kind/required/constraint IDs 不变 |
| food、gourmet 与餐次边界 | 通过 | Stage 6 只检索并标记 food 命中；Stage 7A 才形成 gourmet 候选并执行饮食排除；Stage 7B 只筛选 ID 子集；Stage 8 才分配 `meal_role`、三餐槽位和每天最多一个 gourmet，营业时间与便餐留给 Stage 9A/11/12B |
| 错误与业务不可行分流 | 通过 | 合法空检索、空候选池、gourmet 不可用、required 饮食冲突、selected_count 小于天数和核心 LLM 重试耗尽均有独立状态；核心筛选和 PrimaryDraft 耗尽不得复用 Stage 5 `parse_failed` ignored 例外 |
| Stage 9A 锚点边界 | 通过 | Stage 8 只输出 `DAY_START/LUNCH_SLOT/DAY_END` 或同日已有 POI ID；无下午点且午餐待选时强制 `dinner_after_anchor=LUNCH_SLOT`，不得引用未来酒店或便餐 ID |
| ignored、天气与越界字段 | 通过 | ignored 子句不进入检索、候选、筛选或草案；Stage 18 前天气对 Stage 6～8 决策零影响；6～8 均禁止输出下游的酒店、便餐、路线、时间轴、预算或最终 TripPlan 字段 |
| R2～R10 设计完整性 | 通过 | 四个分册均已列出基线差异、文件范围、输入输出/错误契约、配置/开关、实施步骤、不做事项、自动化测试、人工验收、退出指标、迁移与回滚；Stage 7A 类型不匹配的实施步骤已收口为排除诊断 |

审计结论：Stage 6、7A、7B、8 的 R2～R10 设计材料通过第二次跨 Stage 一致性复核。R1 均保持“执行 Ready 待前置 Stage 实际 Done 后补证”；当前结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步可开始细化 Stage 9A 分册。

### 11.3 第一批 Stage 11→12A→12B→13A 交接审计

审计日期：2026-07-10。审计范围为 Stage 11、12A、12B、13A 分册及其与 Stage 9B/10A/10B、02/03/04 的接口；这是用户批准的临时批处理细化，不改变 `99_todo.md` 的常规“单工作包”规则，也不代表代码、测试、Gate 或设计冻结已完成。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 依赖与顺序 | 通过 | 10B 的 RouteSegment/查询参考进入 11；11 timeline 进入 12A budget，再与预算共同进入 12B；仅 12B valid 才由 13A assemble complete |
| 时间轴边界 | 通过 | 11 只前向排程、报告超窗和一次 transit refresh 请求；不改草案/路线/餐厅/酒店，也不决定 partial/failed |
| 预算真实性 | 通过 | 12A 只使用来源价格或本地明确规则；规则金额为 estimated，缺价/缺规则为 unavailable，known total 不含 unavailable |
| 校验与修复分流 | 通过 | 12B 将来源、路线、时间、三餐、住宿、accepted constraints 与预算输入映射为稳定 issue/scope；不自行修复；13A invalid 固定为 requires_repair |
| 组装与状态 | 通过 | 13A 只从 valid 输入回填来源事实并生成 complete；partial/failed、修复、HTTP 错误和默认切换分别留给 13B～16 |
| API/迁移边界 | 通过 | legacy/shadow 普通请求零新链路；new 仍 503；13A 只允许本地显式验证，线程池不改变默认响应 schema |
| 配置与测试 | 通过 | 12A 新增 `TRIP_TAXI_PRICING_RULES_PATH` 登记；四个分册均具备精确文件表、离线定向/全量/构建命令、证据和逆序回滚 |

审计结论：第一批 Stage 11、12A、12B、13A 的 R2～R10 设计材料通过交接一致性复核。R1 仍必须等待各自前置 Stage 实际 Done 后补证；该结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步细化 Stage 13B。

### 11.4 第二批 Stage 13B→13C→13D 交接审计

审计日期：2026-07-10。审计范围为 Stage 13B、13C、13D 分册及其与 9A～13A、02/03/04 的 repair、状态和重建接口；这是用户批准的临时批处理细化，不改变 `99_todo.md` 的常规“单工作包”规则，也不代表代码、测试、Gate 或设计冻结已完成。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 修复优先级 | 通过 | 13B 只按 route→meal→lodging 替换真实同级事实；未闭合才交 13C，避免局部修复越界改 PrimaryDraft |
| 重排/删除边界 | 通过 | 13C 仅对 explicit-time 进入受限重排；普通超窗最后按最小 optional removal_rank 删除一个点，并从 PrimaryDraft 全量重建 9A～12B |
| 不可变量 | 通过 | accepted constraints、required、来源事实、PrimaryCandidateSelection 与交通偏好贯穿 13B～13D；酒店/三餐/必打卡不得删除 |
| RepairContext | 通过 | 8 次全局、3 次同类、已试 ID 与 canonical fingerprint 在三 Stage 间连续继承；重复状态、无候选或上限只形成业务停止，不伪装技术成功 |
| 全局与前缀收口 | 通过 | 13D 先在完整 N 日范围跨日重排；仅失败后 K=N-1…1 逐级重建连续前缀，禁止裁剪旧数组/跳日；DayK 仍按非末日、K+1 lodging/预算重算 |
| 结果分流 | 通过 | 技术错误原样传播；accepted/food 硬约束不可行直接 failed；complete/partial 只由 Validator.valid 输入组装，degraded 仍不由本批擅自置位 |
| API/迁移与测试 | 通过 | 默认流量仍 legacy/shadow，new 仍 reserved；三分册均有精确文件、离线定向/全量/构建命令、证据和逆序回滚 |

审计结论：第二批 Stage 13B、13C、13D 的 R2～R10 设计材料通过交接一致性复核。R1 仍必须等待各自前置 Stage 实际 Done 后补证；该结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步细化 Stage 13E。

### 11.5 Stage 13E 独立交接审计

审计日期：2026-07-10。审计范围为 Stage 13E 分册及其与 Stage 2、13A、现有 Vue 页面、Stage 14～16、27、32 的前端状态接口；本次为一个独立工作包的 R2～R10 设计复核，不替代 `99_todo.md` 要求的 Stage 9A～16 全范围一致性审计，也不代表代码、测试、Gate 或设计冻结已完成。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 唯一状态责任 | 通过 | Pinia `useCurrentTripStore` 是唯一正式当前 `TripPlan` 入口；Result 不再在 mounted 时持有/恢复正式副本，组件临时地图、图片、折叠和 edit draft 不构成正式事实 |
| API/schema 迁移边界 | 通过 | 13E 继续承载当前 legacy 前端 `TripPlan`，不提前消费 new/shadow 结果或建设双 schema；Stage 16 才同步替换类型、响应解码与 snapshot version |
| 原子性与迟到结果 | 通过 | 单调 request sequence 覆盖 begin/replace/fail/clear；过期或清空后的 success/error 都不能覆盖较新/已清空行程，替换只接受完整 guard 通过的 plan |
| 恢复与存储边界 | 通过 | 仅 versioned sessionStorage snapshot 保存 ready 完整 plan；loading/error/token/draft/raw 数据不保存，旧 key 仅一次性确定性迁移，坏值只清理本 Store key |
| 后续 Stage 接管 | 通过 | 14 负责删除编辑并只读展示，15/16 负责错误/API 状态，27 负责导航，32 扩展 revision 原子替换；13E 不提前实现它们 |
| 验证与回滚 | 通过 | 无新增前端测试框架，采用 typecheck+build 和四个记录化人工状态场景；回滚先逆序下游，且仅在可验证时降写旧 snapshot，禁止全量清 storage 或虚构转换 |

审计结论：Stage 13E 的 R2～R10 设计材料通过独立交接一致性复核。R1 仍必须等待 Stage 13A 实际 Done 后补证；该结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步细化 Stage 14。

### 11.6 第三批 Stage 14→15→16 交接审计

审计日期：2026-07-10。审计范围为 Stage 14、15、16 分册及其与 Stage 13D、13E、02/03/04 的结果状态、前端状态、错误/真实性和 mode 接口；这是用户批准的临时批处理细化，不改变 `99_todo.md` 的常规“单工作包”规则，也不替代 Stage 9A～16 全范围一致性审计。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 状态消费顺序 | 通过 | 13D 先定义可信 complete/partial/failed，14 只读呈现其时间轴、K+1 lodging 与 missing；15 不改业务判定，16 才将正式状态切入默认 API |
| 前端事实边界 | 通过 | 14 删除全部手动编辑与事实字段 mutation；13E Store 原子替换，16 同提交升级为 v2 正式 schema；地图/PDF/对话仍留后续 Stage |
| 业务与技术分流 | 通过 | 15 让业务不可行保留 HTTP 200 TripPlan，配置/超时/Provider/核心输出走稳定 error envelope；parse_failed 是 ignored 例外，fake stale 仅验证边界不抢占 17 |
| 真实性与 fallback | 通过 | 16 删除北京默认坐标、平移、直线、虚构实体/费用和 LLM 补事实；accepted/首日不可行均 failed，legacy 回滚同样禁止恢复虚构分支 |
| API/mode 迁移 | 通过 | 16 在 Gate B 证据齐备后默认 new，API 直出正式 TripPlan；legacy 是短期受控回滚，shadow 普通流量零双跑，前端不建设长期 adapter |
| 测试与回滚 | 通过 | 三分册均有精确文件、离线定向/全量/build 命令、PC/API 人工验收和依赖逆序回滚；new→legacy→new 演练不允许丢失状态或恢复虚构 fallback |

审计结论：第三批 Stage 14、15、16 的 R2～R10 设计材料通过交接一致性复核。R1 仍必须等待各自前置 Stage 实际 Done；Stage 16 的 default new 还必须等待 Gate B 实际证据。该结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步执行 Stage 9A～16 全范围一致性审计。

### 11.7 Stage 9A～16 全范围一致性审计

审计日期：2026-07-10。审计范围为 Stage 9A、9B、10A、10B、11、12A、12B、13A～13E、14、15、16 全部分册及 02/03/04 的契约；本次只验证 R2～R10 设计一致性，不代表代码、测试、Gate 或任一 R1 已通过。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 草案到活动事实链 | 通过 | 9A 将 PrimaryDraft 的 ID/meal slot/anchor 收口为连续 lodging、早餐与真实便餐；9B 只合并白名单 ID 为完整活动顺序，10A/10B 才基于回填坐标查询必要真实路线 |
| 时间、预算与校验闭环 | 通过 | 11 只计算 timeline/buffer 并最多一次公交刷新；12A 仅生成 known total/assumptions/unavailable，12B 统一报告稳定 issue；三者均不擅自修复或组装结果 |
| 修复、partial 与约束 | 通过 | 13B route→meal→lodging 局部替换，13C 重排/删 optional，13D 全局重排后递减前缀；accepted/required/food 不可放宽，partial 是连续完整前缀且 DayK/K+1 lodging/预算一致 |
| 业务、技术与降级分流 | 通过 | 合法空/最终不可行才为 partial/failed；Provider/LLM/schema/config 为 Stage 15 error envelope；parse_failed 是 ignored 例外；stale 只在可信输入时 `degraded=true`，真实缓存仍留 17 |
| 前端与 schema 切换 | 通过 | 13E 是唯一当前行程状态；14 只读时间轴/预算/status 并删除手动编辑；16 同步切换正式 TripPlan、snapshot v2 与 new 默认，failed 不携带 `days`，不建设长期 adapter |
| mode、真实性与回滚 | 通过 | 16 清除默认北京坐标、平移、直线、虚构实体/费用和 LLM 补事实；Gate B 证据前不得 default new；legacy 回滚与 shadow 均不得恢复虚构 fallback 或普通流量双跑 |

审计中发现并已修正两处规格衔接：Stage 15 改用独立技术错误组件，不依赖其未声明的 Stage 14 状态横幅；Stage 16 的 failed 结果改为不携带 `days`，与 Stage 13D 的失败 shape 一致。

审计结论：Stage 9A～16 的 R2～R10 设计材料通过全范围一致性审计。所有 Stage 仍须在其实际前置 Done 后逐项补 R1；Stage 16 的 default new 还须完成 Gate B 实际证据。该结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步细化 Stage 17。

### 11.8 Stage 17 独立设计审计

审计日期：2026-07-11。审计范围为 Stage 17 分册及其与 Stage 1、2、3、4、10A、10B、15、16、02/03/04 的缓存、来源、错误和降级契约；本次只验证 R2～R10 设计一致性，不代表代码、测试、Gate C、设计冻结或任一 R1 已通过。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 依赖与顺序 | 通过 | Stage 17 必须等待 Stage 1 ADR/Provider 结论和 Stage 16 正式 `degraded` schema；Stage 3/4 结构化 AmapService、10A/10B 请求内去重、15 error envelope 均为被消费契约 |
| 缓存边界 | 通过 | 仅缓存已通过领域模型校验的 POI/geocode/weather/route 事实；不缓存 raw payload、LLM 输出、TripPlan、自由文本、Prompt、Key、错误 envelope 或二进制 |
| key 与 Provider 语义 | 通过 | key 包含 operation、标准化参数、Provider、schema/capability version、GCJ-02 与策略；Provider 切换不触发 degraded，也不得误复用不兼容条目 |
| fresh/stale 分流 | 通过 | fresh 命中跳过 Provider；实时成功写 live；只有实时技术失败才尝试可信 stale；无 stale/坏 stale 保持 Stage 15 技术错误，不构造 partial/failed |
| degraded 传播 | 通过 | `cached_stale` 是本 Stage 唯一新增的真实 `degraded=true` 来源；`cached_fresh` 不降级，degraded 不改变 complete/partial/failed、accepted constraints、路线真实性或修复规则 |
| 配置与回滚 | 通过 | 容量、TTL 和 stale 窗口由 Stage 1 ADR 定稿并接入 Settings；内存缓存无持久化迁移，回滚按 degraded 汇总→AmapService 包装→cache/key 模块逆序 |
| 测试与证据 | 通过 | 分册定义 14 个自动化用例、4 个人工场景、缓存键/freshness/调用次数/回滚证据；默认测试离线且使用 FakeAmapProvider 与注入时钟 |

审计结论：Stage 17 的 R2～R10 设计材料通过独立一致性复核。R1 仍必须等待 Stage 1 和 Stage 16 实际 Done，并在开工前复核 Stage 1 ADR 是否已给出 TTL、容量和 stale 窗口的真实证据。该结论不表示 Ready、Done、设计冻结或允许修改工程代码。下一步细化 Stage 18。

### 11.9 Stage 18～20 跨 Stage 一致性审计

审计日期：2026-07-11。审计范围为 Stage 18、19、20 分册及其与 Stage 4、7A、7B、8、10A/10B、13D、14、15、16、17、02/03/04 的天气、地图、Key、来源、错误和 Gate C 契约；本次只验证 R2～R10 设计一致性，不代表代码、测试、Gate C、设计冻结或任一 R1 已通过。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 依赖与批次边界 | 通过 | Stage 18 消费 Stage 4 天气、7A/7B/8 预留和 13D 重建；Stage 19 先于 Stage 20 收口 Web JS Key；Stage 20 消费 10B RouteSegment、14 timeline focus、16 正式 TripPlan 和 17 freshness |
| 天气真实性与决策边界 | 通过 | 天气只使用结构化预报范围内数据；超范围/缺失为 unavailable；室内外标签区分 sourced/estimated/unknown，unknown 不被硬判；Agent 只能替换已有候选 ID |
| 下游重建边界 | 通过 | 天气导致主要点位集合、日期或顺序变化后，必须从 PrimaryDraft 起重建 9A～13D 全部下游；不得复用旧住宿、便餐、路线、时间轴、预算或校验结果 |
| 错误与 degraded 分流 | 通过 | 天气技术错误无可信输入时走 Stage 15 error envelope；合法 unavailable 不阻断规划；cached_stale 天气/路线只触发 degraded，不改变 complete/partial/failed 或 accepted 约束 |
| Key 安全与配置 | 通过 | Stage 19 区分后端 `AMAP_API_KEY` 与前端 `VITE_AMAP_WEB_JS_KEY`，要求 localhost/127.0.0.1 来源限制、配额、最小权限、构建扫描和脱敏证据；Stage 20 只消费配置状态 |
| 地图真实性与联动 | 通过 | Stage 20 只绘制 RouteService 的真实 GCJ-02 polyline；缺失 polyline 显示 unavailable，禁止端点直线；地图与时间轴通过 `day_id + segment_id` 双向联动，不改写 TripPlan |
| 导出技术记录 | 通过 | Stage 20 只验证 html2canvas 捕获能力并输出 Stage 25 输入；捕获失败时使用基于真实 polyline 的 SVG 快照，禁止空白或直线占位冒充成功 |
| 测试、证据与回滚 | 通过 | 三个分册均列出文件范围、输入输出契约、配置/开关、不做事项、自动化/人工验收、量化指标、证据和逆序回滚；默认不引入新前端测试框架作为硬依赖 |

审计结论：Stage 18～20 的 R2～R10 设计材料通过跨 Stage 一致性复核。R1 仍必须等待各自前置 Stage 实际 Done 后逐项补证；Gate C 仍须实施期真实测试、Key 控制台证据、地图人工验收和 Stage 16 默认 new 实际证据。该结论不表示 Ready、Done、Gate C 通过、设计冻结或允许修改工程代码。下一步细化 Stage 21。

### 11.10 Stage 21 独立一致性审计

审计日期：2026-07-11。审计范围为 Stage 21 分册及其与 Stage 19、23、25、26、27、28、02/03/04 的本地代理、请求客户端、地图 Key、图片、PDF 和 SSE 接口边界；本次只验证 R2～R10 设计一致性，不代表代码、测试、Gate D、设计冻结或 R1 已通过。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 依赖与阻断边界 | 通过 | Stage 21 依赖 Stage 19，只统一请求入口和本地代理；未通过时阻断 Stage 23、25、27、28 的前端请求接入 |
| 本地代理范围 | 通过 | 浏览器业务 API 使用相对 `/api`，Vite proxy target 仅限本地开发配置；不设计生产 base URL 或扩大 CORS |
| 硬编码 host 清除 | 通过 | `services/api.ts` 和 Result 图片请求纳入扫描与迁移范围，允许 `vite.config.ts` 保留本地 proxy target |
| 请求类型隔离 | 通过 | JSON client 正式落地，stream、multipart、download 只预留职责边界；避免 JSON header/axios 拦截器污染后续 SSE、PDF 或 Blob |
| 后续 Stage 衔接 | 通过 | Stage 23 图片批量、Stage 25/27 PDF、Stage 28 SSE 都只消费请求边界；Stage 21 不提前新增业务接口 |
| Key 与安全 | 通过 | 沿用 Stage 19 的 `VITE_AMAP_WEB_JS_KEY`，地图 Web JS 不经 `/api` 代理；日志和证据不记录 Key、payload、Blob 或堆栈 |
| 测试、证据与回滚 | 通过 | 覆盖源码/构建扫描、前端 build、dev 代理冒烟、图片路径、地图 Key 回归和回滚证据 |
| 回滚边界 | 通过 | 回滚顺序先恢复调用方再恢复请求模块；禁止回滚为组件内新增硬编码 host |

审计结论：Stage 21 的 R2～R10 设计材料通过独立一致性复核。R1 仍必须等待 Stage 19 实际 Done 后补证；Gate D 仍须后续 Stage 22～30 实施期测试、PDF/图片/SSE/PC 显示真实证据和整体体验闭环。该结论不表示 Ready、Done、Gate D 通过、设计冻结或允许修改工程代码。下一步细化 Stage 22～24。

### 11.11 Stage 22～24 跨 Stage 一致性审计

审计日期：2026-07-11。审计范围为 Stage 22、23、24 分册及其与 Stage 13A、14、21、25、02/03/04 的天级展开、图片批量请求、图片加载终态、导出准备和 Gate D 契约；本次只验证 R2～R10 设计一致性，不代表代码、测试、Gate D、设计冻结或任一 R1 已通过。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 依赖与批次边界 | 通过 | Stage 22 消费 Stage 14 的 day/time 引用；Stage 23 消费 Stage 21 请求入口和 Stage 13A API 基线；Stage 24 依赖 Stage 22 全部展开与 Stage 23 批量结果 |
| Stage 25 输入完整性 | 通过 | Stage 22 提供 expandAll/snapshot/restore；Stage 24 提供 `imagesReady`/`waitForImagesReady()`，共同满足 Stage 25 自动全展开并等待图片终态的前置输入 |
| 展开状态一致性 | 通过 | Stage 22 明确使用稳定 day key，默认全部折叠，支持全部展开/折叠，TripPlan 替换清理旧状态；不修改 TripPlan、时间轴、地图或后端 |
| 图片批量契约 | 通过 | Stage 23 定义 `/api/poi/photos/batch`、client_ref 映射、前后端双重去重、单批上限、单项状态和部分成功；不提前实现懒加载或导出等待 |
| 图片终态与等待 | 通过 | Stage 24 将 loaded/fallback 作为终态，failed 不可无限 pending；慢网、404、超时、重试耗尽都进入 fallback，并提供 ready summary |
| 错误、降级与真实性 | 通过 | 图片 not_found/failed 不写回 TripPlan 事实字段；fallback 只代表 UI 可显示/可导出终态，不冒充真实图片来源成功 |
| 配置、安全与日志 | 通过 | 三个 Stage 不新增用户功能开关；新增常量/env 需登记；证据和日志禁止真实 Key、完整 TripPlan、自由文本和完整图片 URL 列表 |
| 测试、证据与回滚 | 通过 | 覆盖 1～3 天展开、请求数收敛、单项失败、缓存、慢网、404、超时、ready、构建、人工验收和逆序回滚 |

审计结论：Stage 22～24 的 R2～R10 设计材料通过跨 Stage 一致性复核。R1 仍必须等待各自前置 Stage 实际 Done 后逐项补证；Gate D 仍须 Stage 25～30 细化、实施期真实测试、PDF/历史/SSE/PC 显示证据和整体体验闭环。该结论不表示 Ready、Done、Gate D 通过、设计冻结或允许修改工程代码。下一步细化 Stage 25。

### 11.12 Stage 25 独立一致性审计

审计日期：2026-07-11。审计范围为 Stage 25 分册及其与 Stage 20、22、24、27、02/03/04 的地图导出、自动全展开、图片 ready、PDF Blob、分页、下载和 Gate D 契约；本次只验证 R2～R10 设计一致性，不代表代码、测试、Gate D、设计冻结或 R1 已通过。

| 审计维度 | 结论 | 关键收口 |
|---|---|---|
| 依赖与阻断边界 | 通过 | Stage 25 依赖 Stage 20 地图导出记录、Stage 22 展开状态接口和 Stage 24 图片 ready；未通过时阻断 Stage 26/27 |
| 自动全展开与恢复 | 通过 | 导出流程先 snapshot，再在独立导出上下文 expandAll，成功或失败都 cleanup 并 restore 原展开状态 |
| 图片和字体等待 | 通过 | Stage 25 消费 Stage 24 `waitForImagesReady()`，只在 loaded/fallback 后截图；不另建图片状态机 |
| 地图真实性 | 通过 | 优先使用 Stage 20 捕获记录；捕获不稳定时使用真实 RouteSegment polyline SVG 快照；禁止空白地图、端点直线或重新请求路线冒充成功 |
| PDF Blob 边界 | 通过 | 成功导出只生成一个正式 Blob，浏览器下载和 Stage 27 上传必须复用同一 Blob；本 Stage 不接后端保存 |
| 分页与内容完整性 | 通过 | 规定卡片感知 A4 分页，保护标题、天 header、时间轴事件、点位卡片、图片、餐饮和地图，不以整张长图平移作为成功标准 |
| 错误与降级 | 通过 | 任一步失败都不得静默导出缺关键内容的成功 PDF；失败必须清理导出容器、恢复状态并给稳定错误 |
| 测试、证据与回滚 | 通过 | 覆盖三种展开状态、1～3 天、图片/地图失败、分页、单 Blob、异常恢复、构建扫描和逆序回滚证据 |

审计结论：Stage 25 的 R2～R10 设计材料通过独立一致性复核。R1 仍必须等待 Stage 20、22、24 实际 Done 后补证；Gate D 仍须 Stage 26～30 细化、实施期真实测试、PDF 历史/SSE/PC 显示证据和整体体验闭环。该结论不表示 Ready、Done、Gate D 通过、设计冻结或允许修改工程代码。下一步细化 Stage 26～27。

---

## 十二、后续填充顺序

本文后续按以下顺序增量完善，每批独立评审：

1. 建立全局追踪矩阵和测试、配置、功能开关规范；
2. 细化 Stage 1～5，先打通外部事实与核心契约的可开工条件；
3. 细化 Stage 6～16，完成核心规划链路、修复闭环及默认链路切换；
4. 细化 Stage 17～30，覆盖缓存、地图、PDF、SSE 和体验增强；
5. 细化 Stage 31A～33，覆盖无状态修订、对话前端和配置收口；
6. 执行 02→03→04→代码的全盘一致性审计。

当前总纲、统一判定规则和 Stage 1～5 分册已完成首轮跨 Stage 设计审计，Stage 6、7A、7B、8 已完成细化并通过第二次跨 Stage 设计审计，Stage 9A～16 已完成细化并通过全范围一致性审计，Stage 17 已完成独立设计审计，Stage 18～20 已完成细化并通过跨 Stage 一致性审计，Stage 21 已完成细化并通过独立一致性审计，Stage 22～24 已完成细化并通过跨 Stage 一致性审计，Stage 25 已完成细化并通过独立一致性审计；下一步细化 Stage 26～27，不再向总纲堆叠完整工作包。
