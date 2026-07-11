# Stage 14：复合时间轴卡片与移除手动编辑

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 14 |
| 目标 | 将结构化 `TripPlan` 呈现为只读复合时间轴，明确酒店、活动、真实路线、buffer、三餐、预算事实与缺失；彻底移除前端手动改行程入口 |
| 对应优化项 | P0-8 可执行时间轴信息（展示部分）、P0-7 结果消费、P2-1 只读当前行程 |
| 前置依赖 | Stage 13D、13E |
| 后续阻断 | 未通过时不得开始 Stage 20、22、25、29、30；Stage 16 切换时必须复核本页的 complete/partial/failed 消费能力 |
| 默认流量影响 | 不切换 API 或 `TRIP_PLANNER_MODE`；仅在受控新 schema fixture/验证入口和 13E Store 中验证新展示 |

## 2. 当前代码基线与差异

- `Result.vue` 以旧日卡片、景点列表和预算汇总展示 legacy 字段；没有 route/timeline/buffer、起止酒店、餐窗、来源缺失或 partial 语义。
- 地址、游览时长、描述、景点顺序与删除按钮仍可直接编辑并写回当前行程，违反“前端只消费 TripPlan”的边界。
- Stage 11/12A/12B/13D 已定义时间轴、预算、Validator 和 complete/partial/failed 输入；13E 已建立唯一 Store，但 Stage 16 前普通 API 仍是 legacy schema，故不得在本 Stage 建长期 response adapter。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `frontend/src/types/index.ts` | 补齐 Stage 2/13D 的展示只读类型、null/unavailable 与时间轴展示 DTO | 前端不重算时间/预算、不保留 legacy/new 双模型 |
| 修改 | `frontend/src/views/Result.vue` | 只组合 Store、状态横幅、日时间轴、预算与空态；移除编辑/保存/删除/排序逻辑 | 不直接修改 `TripPlan` |
| 新增 | `frontend/src/components/TripTimeline.vue` | 渲染 Day header、活动轨道、路线、buffer、餐饮与日时长 | 每段稳定以 `day_id/segment_id/activity_id` 标识并可 emit focus |
| 新增 | `frontend/src/components/TripBudget.vue` | 渲染已知费用、假设和 unavailable 项 | 不把 known total 表述成完整预算 |
| 新增 | `frontend/src/components/TripStatusBanner.vue` | 预留 complete/partial/failed/degraded 的只读呈现 | 不把 partial/failed 伪装完整成功 |
| 修改 | `frontend/src/views/Home.vue` | 移除“结果可手工编辑”的文案/跳转假设，保留生成入口 | 不改请求协议或错误分类 |
| 新增 | `doc/opt_strategy/evidence/stage_14/` | 保存 build、PC 人工验收、只读扫描和回滚摘要 | 仅脱敏文字/必要截图 |

本 Stage 不实现真实地图、图片/导出/PDF、折叠策略、API 错误分类、请求重试或新链路默认切换；分别由 Stage 15、16、20、22～25 负责。

## 4. 输入、输出与展示契约

### 4.1 时间轴

- 每日以 `start_hotel → timeline events → end_hotel/null` 呈现。非末日最后一段必须为最后活动到 `end_hotel` 的真实路线；末日以晚餐结束，不显示虚构结束酒店。
- 活动卡显示出发、到达、离开时刻；路线卡显示方式、距离、真实耗时与必要换乘摘要；`buffer` 是独立轨道段，显示缓冲时长，不能并入路线或景点时长。
- 景点与美食级餐厅同级显示；美食级餐厅依其 `meal_role` 进入早餐/午餐/晚餐窗口，且同餐次不得再显示便餐。便餐午晚餐既是时间轴正式活动，也在底部餐饮条汇总具体高德店名/分店、地址与角色。
- 非美食早餐固定显示“酒店餐厅或附近早餐店”，并显示酒店政策/饮食未核实提示；不得伪造 POI、路线、营业时间或费用。营业时间 unavailable 显示“营业时间待核实”，其他事实缺失统一显示“暂无数据”，不得显示 `0`、空字符串或猜测值。
- 组件只格式化带 `+08:00` 的 ISO 时间和 Decimal/CNY 显示，不推算 arrival/departure、日时长、路线、meal window 或 buffer；非法/未知枚举按稳定“暂无数据”降呈现并记录脱敏开发诊断。
- `TripTimeline` 可发出 `{ day_id, segment_id }` focus 事件供 Stage 20 接地图；本 Stage 不创建地图、polyline 或直线替代。

### 4.2 酒店、预算与状态

| 区域 | 必须显示 | 禁止 |
|---|---|---|
| Day header | 日期、Day 序号、`start_hotel`、非末日 `end_hotel`；酒店不同明确“换酒店” | 将 DayK partial 当末日或凭空补酒店 |
| 预算 | 2 名成人、1 间房、实际 `lodging_count` 晚；公共交通按人、taxi 按车；逐项来源/estimated/unavailable | 将 unavailable 加入 total，称 known total 为“总预算” |
| complete | 完整计划标识与所有 N 日 | 隐藏 missing/unavailable 提示 |
| partial | “已生成连续 Day1..DayK”、首个失败日及后续截断说明 | 显示 K 后日期、称完整成功、按 K 晚而非 K+1 晚展示住宿 |
| failed | 仅失败摘要、稳定原因码和返回首页/重新生成入口 | 渲染 days、旧 plan、技术堆栈或虚构内容 |
| degraded | 与 plan_status 独立的来源/时效提示 | 以成功样式掩盖 stale，或将其改写为 failed |

Stage 14 的 fixture 可以覆盖 13D 已定义的三种 status；正式 API 形状仍由 Stage 16 一次性接入。13E Store 在 16 时升级 snapshot guard，不能由本 Stage 建永久 legacy/new 转换。

### 4.3 只读不变量

- 删除 `editMode`、`originalPlan`、`toggleEditMode/saveChanges/cancelEdit/deleteAttraction/moveAttraction` 以及所有 `v-model` 到 TripPlan 事实字段的入口；地图、图片和导出也不得借回调改写 plan。
- 所有展示组件接收 readonly props；组件局部状态只可为折叠/hover/focus 等 UI 状态，不得克隆为可保存的正式 `TripPlan`。
- 对话式变更仅能在 Stage 32 经后端重算并由 Store 原子替换；本 Stage 的页面只能查看、返回首页或清空当前会话。

## 5. 配置与功能开关

不新增配置、环境变量或功能开关。沿用 13E 的 Store/snapshot 与 `TRIP_PLANNER_MODE` 语义；默认 legacy/shadow 流量不得被本 Stage 双跑新链路。

## 6. 实施步骤

1. 固定只读前端展示 DTO 与 null/unavailable/status 格式化边界。
2. 拆出时间轴、预算和状态组件，使用完整/partial/failed/degraded fixture 验证结构。
3. 将 Result 改为 Store 驱动的复合时间轴，保留 Stage 20 所需 focus 事件而不接地图。
4. 删除全部手动编辑与事实字段 `v-model`，复核组件没有写 Store plan 的路径。
5. 在 PC 主流宽度下完成 build、只读扫描与人工验收，归档证据。

## 7. 明确不做

- 不计算或修复路线、时间、预算、status、missing items 或来源质量；只展示 TripPlan。
- 不实现地图真实路线、图片加载、全部展开、PDF、历史、SSE、导航守卫或响应错误分类。
- 不在前端使用 0/直线/默认酒店/生成文案补全缺失事实，不恢复任何手动编辑入口。

## 8. 自动化测试

当前不新增前端测试框架；以严格类型检查和生产构建为自动化验收，UI 语义使用 fixture 的记录化人工验收。

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S14-T01 | build | 时间轴、预算、status 组件类型检查 | readonly DTO、null/status 分支与 emit 均可编译 |
| S14-T02 | build | 只读扫描 | 不再存在 TripPlan 的 edit/save/delete/move 或事实字段 `v-model` |

```text
npm --prefix frontend run build
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S14-M01 | 以 complete fixture 查看 1 日与 3 日计划 | 起止酒店、真实路线、buffer、三餐与日时长顺序清楚 |
| S14-M02 | 以 partial K=2/N=3 fixture 查看 | 仅 Day1..2，Day2 按非末日到 lodging[2]，预算显示 3 晚与截断说明 |
| S14-M03 | 以 unavailable 营业时间/票价、lodging-side 早餐 fixture 查看 | 显示核实/暂无数据，不出现 0 或虚构店名 |
| S14-M04 | 检查结果页全部交互 | 无编辑、保存、删点、改时长、改地址或排序入口；PC 无横向截断 |

## 10. 量化退出指标

- [ ] S14-T01～T02、S14-M01～M04 通过，构建退出码为 0。
- [ ] TripPlan 事实字段的可写 UI 入口、组件直接 mutation、直线/默认值补事实次数均为 0。
- [ ] complete/partial/failed/degraded 误呈现为完整成功的次数为 0；partial K+1 lodging/预算误示次数为 0。
- [ ] unavailable 被显示为 0/空白而未提示的次数为 0；PC 主流宽度信息截断数为 0。

## 11. 迁移与回滚

- 迁移仅替换 Result 的展示组件和删除编辑能力，不改变 Store snapshot、API 或后端数据。
- 回滚先回滚后续 16/15/20 等依赖，再恢复原 Result 展示；不得恢复直接改事实的编辑按钮，也不得将 partial/failed 伪装 complete。
- 回滚后运行 build 和 M01～M03，保留 13E 唯一 Store 边界。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_14/test-summary.md` | 提交、构建、2 用例、失败数 |
| 展示矩阵 | `doc/opt_strategy/evidence/stage_14/timeline-status-matrix.md` | complete/partial/failed/degraded/unavailable 与测试 ID |
| 人工验收 | `doc/opt_strategy/evidence/stage_14/manual-check.md` | M01～M04、PC 宽度和结论 |
| 回滚验证 | `doc/opt_strategy/evidence/stage_14/rollback-check.md` | 逆序、命令、结果与无编辑确认 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批 Stage 14～16 交接审计；执行 Ready 待 R1**。R1 必须等待 Stage 13D、13E 实际 Done，并复核正式 TripPlan 展示 DTO、partial DayK/K+1 lodging、Store 只读边界和 Stage 16 schema 切换。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
