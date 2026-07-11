# Stage 13E：Pinia 当前行程状态基础

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 13E |
| 目标 | 建立唯一、类型受控且可在当前标签页恢复的前端 `TripPlan` 运行时状态入口；一次请求只可原子替换一次当前行程，旧响应和组件局部副本不得覆盖它 |
| 对应优化项 | P2-1 当前行程导航与 PDF 历史（前端状态基础）；为 P0-8 展示、P1-2 错误呈现及后续修订的原子替换提供边界 |
| 前置依赖 | Stage 13A（最终 `TripPlan` 组装契约）；Stage 2 的领域字段/空值/状态契约在实现时一并复核 |
| 后续阻断 | 未通过时不得开始 Stage 14、15、27、32；Stage 16 切换新 API 前必须复核本 Store 的替换与恢复契约 |
| 默认流量影响 | 不切换 `/api/trip/plan`、不改变 `TRIP_PLANNER_MODE`；仅把现有页面的当前行程内存与 sessionStorage 责任收敛到 Store |

## 2. 当前代码基线与差异

- `main.ts` 只安装 Router 与 Ant Design Vue，尚未安装/挂载 Pinia；`Home.vue` 自己维护提交 loading，并直接写入 `sessionStorage['tripPlan']`。
- `Result.vue` 在 mounted 时自行读取该 key 并复制到局部 `ref`；编辑、取消编辑和保存可直接改该对象。刷新恢复、路由进入和提交完成没有共同的状态机，迟到响应也没有统一的覆盖保护。
- 当前前端 `TripPlan` 类型仍是 legacy 响应形状；Stage 16 才会同步切换普通 API 到 Stage 2/13A～13D 的正式 schema。13E 不能为了 Store 提前让浏览器消费未启用的新链路结果，也不能建设长期双 schema 适配层。
- 当前页面图片缓存、地图实例、折叠面板、导出 DOM 和编辑草稿都混在 `Result.vue`。其中只有“正式当前行程”应进入本 Stage 的 Store；其余 UI 临时状态不应被持久化或提升为全局事实。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `frontend/package.json`、`frontend/package-lock.json` | 引入 Pinia 并锁定与当前 Vue 3 兼容的版本 | lockfile 与 manifest 一致，生产构建可解析单一 Pinia 实例 |
| 修改 | `frontend/src/main.ts` | 创建并安装唯一 Pinia 实例；在挂载应用前执行一次受控 restore | Router/Antd 顺序不受影响，首次渲染前状态可用 |
| 新增 | `frontend/src/stores/current_trip.ts` | 定义 `useCurrentTripStore`、请求序号、原子替换、失败、清空、恢复和最小 sessionStorage 序列化 | 页面无权绕过 action 读写正式行程 |
| 修改 | `frontend/src/types/index.ts` | 保持当前 legacy `TripPlan` 类型为 13E 的唯一承载类型，并收口 Store 状态/错误/快照类型 | 不复制后端内部草案、Prompt、payload 或诊断；Stage 16 有唯一 schema 切换点 |
| 修改 | `frontend/src/views/Home.vue` | 将提交逻辑接入 Store 的 request token、loading/error 与 `replacePlan`；删除直接 sessionStorage 写入 | 仅当前请求可提交结果，失败不抹掉已存在行程 |
| 修改 | `frontend/src/views/Result.vue` | 从 Store 读取正式行程；删除 mounted 直接读写 sessionStorage 和正式行程局部副本 | 无行程时显示空态；临时地图/图片/折叠状态仍可局部保存 |
| 新增 | `doc/opt_strategy/evidence/stage_13e/manual-check.md` | 记录四个前端人工验收场景、浏览器/环境和结论 | 不包含完整行程、token、自由文本或截图中的敏感信息 |
| 新增 | `doc/opt_strategy/evidence/stage_13e/test-summary.md`、`rollback-check.md` | 记录 build、类型检查与回滚验证摘要 | 命令、提交标识、结果和未关闭问题齐全 |

本 Stage 不修改后端、API response schema、`TRIP_PLANNER_MODE`、Router 守卫、PDF/历史、SSE、地图、图片接口或对话接口；它也不删除手动编辑功能，Stage 14 才移除该 UI 能力。

## 4. 输入、输出与错误契约

### 4.1 唯一当前行程状态

Store 名称固定为 `useCurrentTripStore`；正式状态仅由它持有：

| 字段 | 类型/取值 | 语义 |
|---|---|---|
| `tripPlan` | `TripPlan \| null` | 当前标签页唯一正式行程；只读暴露给组件，禁止深层原地改写 |
| `phase` | `idle \| loading \| ready \| error` | 当前请求/恢复后的运行时阶段；`ready` 必须有 plan，`idle` 必须无 plan |
| `error` | `CurrentTripError \| null` | 脱敏、可展示的本次请求失败摘要；不得包含 Axios 原始对象、堆栈或后端完整 details |
| `activeRequest` | 单调递增的内存 request sequence | 仅用于同标签页乱序防护；不写 sessionStorage，不替代后端 `request_id` |
| `restored` | boolean | 应用启动时只执行一次恢复；避免路由组件重复反序列化 |

- `TripPlan` 在 13E 继续表示当前 legacy JSON 的前端承载类型，且只能来自现有成功响应或本地受控 fixture；不得把 `ItineraryDraft`、Agent 文本、`CoordinationOutcome`、内部 repair 诊断或半成品放进 Store。
- Stage 16 对普通 API 同步切换正式 schema 时，原位替换 `TripPlan` 类型、响应解码与 snapshot 版本；不得保留 `legacyPlan` 与 `newPlan` 两个长期并行的 Store 字段或响应适配器。
- `tripPlan` 可在 `loading/error` 阶段保留上一份 valid plan，供用户返回查看；`hasCurrentTrip` 只由 `tripPlan !== null` 推导，不能从 `phase` 推断。组件必须同时按 `phase` 和 `hasCurrentTrip` 渲染。

### 4.2 Store action 与原子替换

```text
Home submit
  → beginRequest() => sequence
  → generateTripPlan(...)
  → replacePlan(sequence, response.data)  # 仅当前 sequence 可提交
  → sessionStorage snapshot
  → Result 从 Store 读取
```

| Action | 前置/输入 | 成功效果 | 拒绝或失败效果 |
|---|---|---|---|
| `restoreCurrentTrip()` | 应用启动、尚未 restored | 解析并校验 snapshot 后一次性置入 `tripPlan/phase/error` | 无 key 或无效数据进入 `idle`；只删除本 Store 的坏 key，不伪造 plan |
| `beginRequest()` | 任意阶段 | 递增 sequence，`phase=loading`、`error=null`，返回 sequence；保留旧 valid plan | 无网络操作、无 storage 写入 |
| `replacePlan(sequence, plan)` | `sequence === activeRequest`，且 `plan` 通过当前前端最小 shape guard | 在同一 action 内置入完整 plan、`phase=ready`、`error=null`，写入完整 snapshot | sequence 过期或 plan 非法时不改变正式行程；过期结果静默丢弃，非法输入转为稳定本地错误 |
| `failRequest(sequence, error)` | `sequence === activeRequest` | `phase=error` 并保存脱敏错误，保留旧 valid plan | 过期错误静默丢弃；不得清空或覆盖当前 plan |
| `clearCurrentTrip()` | 用户明确清空当前会话 | 递增 sequence 后置 `tripPlan=null/phase=idle/error=null`，仅删除本 Store key | 迟到 success/error 因 sequence 不匹配均不得复活已清空行程 |

- 原子替换指组件在一个状态提交中只能观察到“旧完整 plan”或“新完整 plan”，绝不能观察到 days、budget、来源或状态被逐项替换的中间对象。禁止 `Object.assign`、`push/splice` 或组件直接改写 `store.tripPlan`。
- 现有编辑 UI 在 Stage 14 移除前可以持有独立的 `editDraft`，但它不是正式行程；保存时必须经 `replacePlan` 传入完整副本，取消时仅丢弃 draft。不得再次建立 `originalPlan`/`tripPlan` 的正式持久副本。
- `loadingProgress`、加载文案、展开面板、地图实例、图片 URL cache 与导出容器可以留在组件；它们均不写入 Store 或 snapshot，且不能改变事实字段。

### 4.3 sessionStorage 快照与恢复

- key 固定为 `travel-agent.current-trip.v1`，值为 `{ snapshot_version: 1, trip_plan: TripPlan }`；只在 `ready` 的完整 plan 替换后写入。`loading`、`error`、请求序号、编辑 draft、图片、地图与导出状态一律不持久化。
- 写入/读取前使用当前 `TripPlan` 的最小 runtime guard：snapshot/version、城市、日期、days 及其必要容器字段必须成立；类型断言或 `JSON.parse` 成功都不构成有效 plan。Stage 16 如有破坏性 schema 变化，递增 snapshot version，并只迁移可确定验证的旧快照。
- 初次 restore 仅在新 key 缺失时尝试读取现有 `tripPlan` legacy key；验证通过后先写新 key 再删除旧 key。旧值无效时仅删除该旧 key。迁移不读取、写入或清空其他站点/功能的 sessionStorage。
- `sessionStorage` 被浏览器禁用、配额不足或写入失败时，内存中的 valid plan 仍保留，并仅在开发日志/人工证据中记录脱敏 `storage_unavailable`；它不得改写本次规划为失败、也不得回退到 localStorage、cookie、数据库或虚构数据。
- 恢复只面向当前标签页临时连续性，不提供跨标签页同步、历史列表、服务端保存、恢复 PDF 或恢复任意已失效的请求。

### 4.4 页面与错误边界

- `Home.vue` 的网络提交必须取得 Store sequence；axios 成功但 `success=false`、空 data、shape 不合法或抛出异常均通过 `failRequest` 进入前端稳定错误，不直接操作 `Result.vue` 或 sessionStorage。
- `Result.vue` 只使用 Store 的 `tripPlan/phase/error/hasCurrentTrip`。无当前行程时显示可返回首页的空态；本 Stage 不新增 Router guard，Stage 27 才定义“直接访问行程页返回首页”的导航规则。
- 本 Stage 只把已有调用错误收口为脱敏文本/稳定本地 code，例如 `request_failed`、`invalid_plan_response`；`storage_unavailable` 仅为不改变当前 plan 的本地持久化记录。不解释后端 422/502/503/504、业务 `partial/failed` 或重试策略，后者由 Stage 15/16 接管。
- 日志若保留，仅允许记录本地 request sequence、稳定 code、phase、是否有 current plan 和耗时；不得记录完整 plan、地址、坐标、自由文本、plan token、axios payload 或后端堆栈。

## 5. 配置与功能开关

- 不新增环境变量、Settings 或用户功能开关；Pinia 是前端运行时依赖，不读取 `VITE_` 或后端 secret。
- `TRIP_PLANNER_MODE=legacy|shadow|new` 语义保持不变。13E 不让 `new` 可用、不触发 shadow 双跑，仍由现有普通成功响应或受控本地 fixture 驱动 Store。
- sessionStorage key/version 属于内部迁移格式，不是可配置项。它的引入值为 `v1`、默认启用、无单独开关；在 Stage 16 schema 切换时复核/升级，在 Stage 33 复核是否清理旧迁移分支。

## 6. 实施步骤

1. 安装 Pinia，并在 `main.ts` 创建唯一 Pinia 实例、在 app mount 前调用一次 restore。
2. 定义当前 `TripPlan` 承载类型、snapshot/错误类型和最小 runtime guard；固定 key、version 与 legacy key 的一次性迁移规则。
3. 实现 request sequence、begin/replace/fail/clear/restore actions，保证过期响应和清空后的迟到结果都不能覆盖当前状态。
4. 将 `Home.vue` 的提交 loading/error/成功提交接入 Store，删除其直接 sessionStorage 写入；保留本 Stage 之外的展示文案和模拟进度行为。
5. 将 `Result.vue` 改为只读 Store 的正式行程，删除其 mounted storage 读写；将现存编辑态降为可丢弃局部 draft，并确保保存只能提交完整副本。
6. 执行类型检查/生产构建及四场景人工验收，归档 snapshot、乱序防护、清空和回滚记录。

## 7. 明确不做

- 不提前消费 Stage 13A～13D 的新 API response、切换 `/api/trip/plan`、实现 response adapter、改后端 schema 或改变 `TRIP_PLANNER_MODE`；这些在 Stage 16 统一完成。
- 不重做时间轴、预算、partial/failed 卡片、错误分级、重试 UI、地图、图片、SSE、PDF 导出/历史、导航守卫或 PC 布局。
- 不删除手动编辑入口、也不让其直接修改正式 `TripPlan`；删除能力归 Stage 14，后续修订归 Stage 31A～32。
- 不持久化错误、loading、自由文本、agent/raw payload、token、完整诊断、编辑草稿或任何非当前完整 plan；不使用 localStorage 充当历史库。

## 8. 自动化测试

当前前端未引入组件/状态测试框架。本 Stage 不为单一 Store 预装 Vitest 等平台；以严格 TypeScript 检查和生产构建作为自动化验收，并将状态分支以记录化人工验收覆盖。

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S13E-T01 | build | Pinia 安装、类型与生产构建 | `vue-tsc` 与 Vite 均通过，依赖锁定无重复运行时实例 |
| S13E-T02 | build | Home/Result 使用 Store | 旧 sessionStorage 直接读写和 Result 正式 `TripPlan` 局部 ref 不再通过类型/代码复核 |

```text
npm --prefix frontend run build
```

命令期望退出码为 `0`，不访问真实高德、LLM 或新增后端接口。Stage 13E 必测自动化用例数为 2。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S13E-M01 | 以当前 legacy 正常生成一份行程并跳转结果页 | Store 一次完整替换，结果页从 Store 显示；浏览器中仅存在 versioned 当前行程 snapshot |
| S13E-M02 | 在结果页刷新当前标签页 | Store 在 app mount 前恢复同一完整 plan；不重复请求、不显示半份行程 |
| S13E-M03 | 清除/构造无效当前行程 snapshot 后进入结果页 | 无异常或虚构数据；显示无当前行程空态且只移除本 Store 的坏 key |
| S13E-M04 | 发起请求后立即清空或发起第二次请求，再让第一次响应返回 | 已清空或较新的 plan 保持不变；过期 success/error 不能覆盖 Store 或写回 snapshot |

实际记录进入 `doc/opt_strategy/evidence/stage_13e/manual-check.md`。人工记录只保存状态、稳定 code 与结论，不保存完整行程内容或浏览器 storage 导出。

## 10. 量化退出指标

- [ ] S13E-T01～T02、S13E-M01～M04 全部通过，构建命令退出码为 `0`。
- [ ] 正式 `TripPlan` 的组件级 sessionStorage 直接读写数为 0；除 Store 外的正式当前行程副本数为 0。
- [ ] 过期 sequence、清空后迟到 success/error 覆盖当前 plan 的次数为 0；组件可观察到半成品 plan 的次数为 0。
- [ ] 合法 snapshot 刷新恢复成功率为 100%；无效 snapshot 产生虚构 plan、未捕获 JSON 异常或跨 key 清理的次数为 0。
- [ ] loading/error/draft/token/raw payload 被写入 snapshot 的次数为 0；敏感日志/证据命中数为 0。
- [ ] Stage 13E 不改变普通 `/api/trip/plan` 模式、后端外部调用数或前端 API schema；现有页面基本生成路径不回退。

## 11. 迁移与回滚

- 迁移时由 Store 在首次 restore 将合法 legacy `sessionStorage['tripPlan']` 写为 versioned snapshot 后再删除旧 key；完成后所有页面只能通过 Store 访问正式当前行程。
- 回滚前必须先回滚依赖它的 Stage 32、27、16、15、14（若已实施）。随后在仍兼容 legacy 前端 `TripPlan` shape 时，将合法 v1 snapshot 确定性降写回旧 `tripPlan` key，再移除 Pinia 挂载、Store 和页面接入；无效/未知版本只回到无当前行程，绝不猜测转换。
- 回滚只删除本 Stage 的 key/依赖，不得 `sessionStorage.clear()`，不得删除 PDF、历史、后端数据或恢复虚构 fallback。回滚后重新运行构建和 M01～M03，记录在 `rollback-check.md`。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_13e/test-summary.md` | 提交标识、构建命令、2 用例、离线声明、失败数 |
| 状态/快照矩阵 | `doc/opt_strategy/evidence/stage_13e/store-snapshot-matrix.md` | phase、has plan、snapshot、sequence、过期结果与测试 ID |
| 人工验收 | `doc/opt_strategy/evidence/stage_13e/manual-check.md` | S13E-M01～M04 的环境、步骤、实际状态和结论 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_13e/rollback-check.md` | 依赖逆序、key 降写条件、构建/人工命令和结果 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并完成与 Stage 13A、14～16、27、32 的独立交接审计；执行 Ready 待 R1**。R1 必须等待 Stage 13A 实际 Done，并在开工前复核当前 `TripPlan` JSON 承载类型、Pinia/Vue 版本、legacy 响应保持期、sessionStorage guard 和 Stage 14 的只读接管边界。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
