# Stage 20：地图真实路线与时间轴联动

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 20 |
| 目标 | 将结果页地图改为只绘制 RouteService 提供的真实路线 polyline，并与 Stage 14 复合时间轴按稳定 segment ID 双向联动 |
| 对应优化项 | P1-4 地图信息增强，并完成 Gate C 中地图路线真实性、前后端 Key 隔离消费和导出地图技术记录 |
| 前置依赖 | Stage 10B、14、16、17、19；开工前复核 RouteSegment、TripPlan 正式 schema、时间轴 focus 事件、地图 Key 配置和 degraded/status 展示 |
| 后续阻断 | 未通过时不得宣告 Gate C；不得开始依赖地图捕获记录的 Stage 25 或地图历史体验相关工作 |
| 默认流量影响 | 普通结果页地图展示升级；不改变 TripPlan、路线选择、时间轴计算、API schema 或规划结果 |

## 2. 当前代码基线与差异

- 当前 `frontend/src/views/Result.vue` 内聚了地图初始化、marker、polyline、PDF 截图等逻辑；路线可能由景点坐标顺序连线，不能证明来自 RouteService。
- Stage 10A/10B 已规划 `RouteSegment`，每条路线有 route ID、requirement/segment ID、实际方式、distance、duration、GCJ-02 polyline、来源策略和查询参考。
- Stage 14 已规划 `TripTimeline` 发出 `{day_id, segment_id}` focus 事件，但不创建地图；Stage 20 接管地图展示和双向高亮。
- Stage 19 必须先完成 Web JS Key 风险治理；Stage 20 只消费 `VITE_AMAP_WEB_JS_KEY` 的配置状态，不扩大 Key 暴露。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 新增 | `frontend/src/components/TripMap.vue` | 独立地图组件，渲染酒店、活动、分天路线和交互高亮 | 不改写 TripPlan，不生成路线 |
| 修改 | `frontend/src/components/TripTimeline.vue` | 发出/消费 `focusSegment(day_id, segment_id)` 与 hover 状态 | 时间轴与地图稳定互联 |
| 修改 | `frontend/src/views/Result.vue` | 移除旧内联地图逻辑，接入 TripMap；保留无 Key/无路线降级 | 页面结构清晰，旧直线逻辑删除 |
| 修改 | `frontend/src/types/index.ts` | 补齐前端 RouteSegment、MapPoint、MapPolyline 只读类型 | 与 Stage 16 正式 TripPlan 对齐 |
| 新增 | `frontend/src/utils/mapProjection.ts` 或等价工具 | 将 GCJ-02 polyline 转为高德地图可消费路径与 SVG 快照输入 | 不抽稀、不补直线、不改坐标系 |
| 新增 | `frontend/src/utils/exportRouteSnapshot.ts` | 生成基于真实 polyline 的导出 SVG 路线快照输入/能力探测结果 | Stage 25 可复用，不画点位直线 |
| 新增 | `frontend/src/components/__tests__/TripMap.*` 或人工记录 | 覆盖路线缺失、polyline、高亮和无 Key 状态 | 不引入真实高德 Key |
| 新增 | `doc/opt_strategy/evidence/stage_20/` | 保存地图真实性、联动、导出捕获和回滚证据 | 脱敏，不含 Key 或完整坐标列表 |

若当前前端没有测试框架，本 Stage 采用 `npm run build`、类型检查和记录化人工验收；不强制引入新框架。

## 4. 输入、输出与地图契约

### 4.1 地图输入

TripMap 只接受正式 TripPlan 派生的只读视图：

| 输入 | 来源 | 约束 |
|---|---|---|
| `day_id` | TripPlan day | 稳定，和时间轴一致 |
| `segment_id` | RouteSegment | 每日内唯一，地图/时间轴联动唯一键 |
| `origin_ref/destination_ref` | RouteRequirement/活动/酒店引用 | 只用于定位 marker 和标签，不生成路线 |
| `polyline` | RouteService/AmapService | GCJ-02，至少两个点；缺失时显示不可用，不画直线 |
| `actual_mode` | RoutePolicy/RouteSelection | 决定路线样式和图例，不改变方式 |
| `source/freshness` | RouteSegment source metadata | stale 时配合 degraded/status 提示 |

- 地图不得从景点数组、酒店坐标或 DOM 顺序自行推导路线。
- 地图不得调用 AmapService、RouteService、规划 API 或 LLM。
- 地图交互只改变 UI focus，不写 Store 的 TripPlan 事实字段。

### 4.2 展示规则

- 分天路线使用稳定颜色；同一天内不同 segment 使用同色不同透明度/高亮状态。
- 起始酒店、结束酒店、活动点和天序号 marker 均来自 TripPlan 事实 ID；酒店更换时展示 DayN 末段到新酒店和 DayN+1 从同一酒店出发。
- polyline 缺失、非法或 unavailable：该 segment 显示“路线不可用”状态，并在地图/时间轴保持可追溯占位；禁止以端点直线、算法平移或估算路线替代。
- complete/partial/failed：failed 无 days 时不渲染地图；partial 只渲染返回的连续前缀及其真实路线。

### 4.3 双向联动

| 触发 | 行为 |
|---|---|
| 时间轴 hover segment | 地图高亮对应 polyline 和端点 |
| 时间轴 click segment | 地图定位到对应 polyline bounds |
| 地图 hover polyline | 时间轴高亮对应 segment |
| 地图 click polyline/marker | 时间轴滚动/定位到对应 day/segment |

- 联动键固定为 `day_id + segment_id`；不使用数组下标作为唯一来源。
- 迟到渲染、地图加载失败或组件卸载后，不得覆盖当前 Store 或触发规划重试。

### 4.4 导出地图技术记录

- 本 Stage 必须验证当前 html2canvas 对高德 WebGL/Canvas 与跨域瓦片的捕获能力，并记录结论供 Stage 25 使用。
- 若真实地图无法稳定捕获，导出容器使用独立 SVG 路线快照：按 RouteSegment 真实 polyline 绘制酒店、点位、天序号和交通方式。
- 无底图 SVG 必须明确标注为“无底图真实路线示意”；禁止以点位直线、空白占位或截图失败但仍标记成功。

## 5. 配置与功能开关

不新增配置和功能开关。沿用 Stage 19 的 `VITE_AMAP_WEB_JS_KEY`。

- 缺失 Web JS Key：TripMap 显示地图不可用；时间轴、预算、状态和非地图导出不受影响。
- Key 加载失败：只影响地图组件，记录脱敏错误 code，不暴露 Key 或完整异常。

## 6. 实施步骤

1. 从正式 TripPlan 类型中抽取只读地图视图模型，锁定 day/segment/route/marker 映射。
2. 拆出 TripMap，删除 Result 中旧的景点直线 polyline 和地图事实 mutation。
3. 实现真实 polyline、分天颜色、酒店起终点、天序号 marker 和 route unavailable 状态。
4. 接入 TripTimeline 双向 focus 事件，处理 hover/click、地图加载中/失败、partial/failed/degraded 状态。
5. 实现/记录 html2canvas 捕获验证与 SVG 真实路线快照方案，输出 Stage 25 所需能力记录。
6. 完成前端 build、人工 PC/移动视口检查、Key 缺失检查和回滚证据。

## 7. 明确不做

- 不重新计算路线、不请求新路线、不改变路线选择、不以直线或估算路线兜底。
- 不实现 PDF 导出完整重构、自动全展开、图片批量/懒加载、SSE、历史导航或对话修订。
- 不改变后端 TripPlan schema、AmapService、RouteService、TimelineCalculator 或 BudgetCalculator。
- 不在地图交互中修改 TripPlan、accepted constraints、时间轴、预算或 Store 正式事实。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S20-T01 | unit/type | TripPlan → MapView 映射 | day_id/segment_id/marker/route 稳定 |
| S20-T02 | unit | polyline 缺失/非法 | 显示 unavailable，不画直线 |
| S20-T03 | unit | partial/failed | partial 只渲染前缀；failed 不渲染地图路线 |
| S20-T04 | unit | focus 事件 | 时间轴与地图按 day_id+segment_id 双向高亮 |
| S20-T05 | frontend | 缺失 Web JS Key | 地图降级，页面不崩溃 |
| S20-T06 | frontend | 旧直线逻辑扫描 | 不存在从景点坐标直接连线的可达路径 |
| S20-T07 | regression | degraded/stale route | 显示时效提示，不改变路线或 plan_status |
| S20-T08 | export tech | html2canvas/SVG 快照能力 | 成功记录捕获结果；失败时 SVG 使用真实 polyline |

```text
npm --prefix frontend run build
python -m pytest backend/tests -q
rg -n "new AMap\\.Polyline|Polyline|straight|直线|attractions.*location" frontend/src
```

扫描命令在实施时需结合最终代码人工判读：保留真实 RouteSegment polyline 的 `AMap.Polyline`，删除景点端点直线兜底。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S20-M01 | 打开含多天、多路线、换酒店 fixture | 分天颜色、酒店起终点、天序号正确 |
| S20-M02 | hover/click 时间轴和地图路线 | 双向高亮/定位一致，无错段 |
| S20-M03 | 注入缺失 polyline segment | 显示不可用，不出现端点直线 |
| S20-M04 | 缺失/错误 Web JS Key | 地图降级，其余结果页可读 |
| S20-M05 | 执行导出地图捕获探测 | 记录 html2canvas 可用性或真实 polyline SVG 快照方案 |

## 10. 量化退出指标

- [ ] S20-T01～T08、S20-M01～M05 通过，命令退出码为 0。
- [ ] 地图上由景点/酒店端点直接连线的 segment 数为 0；缺失 polyline 被画成直线的次数为 0。
- [ ] 地图/时间轴联动错配、下标漂移、partial 越界渲染和 failed 渲染路线次数均为 0。
- [ ] 地图交互修改 TripPlan、触发重新规划或清空 Store 的次数为 0。
- [ ] 证据、日志、构建产物中真实 Key 和完整坐标列表命中数为 0。

## 11. 迁移与回滚

- 迁移顺序：MapView 类型 → TripMap → Result 接入 → Timeline focus → 导出能力记录；不得先删除旧地图后才建立无 Key/无路线降级。
- 回滚先恢复 Result 对地图组件的调用边界，再移除 TripMap 和 focus 扩展；不得恢复景点直线兜底为“成功路线”。
- 回滚后运行 S20-T02～T06、前端 build、M03～M04，并确认 Stage 14 只读时间轴仍可独立显示。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_20/test-summary.md` | 命令、8 用例、扫描说明、失败数 |
| 地图真实性矩阵 | `doc/opt_strategy/evidence/stage_20/route-map-matrix.md` | RouteSegment、polyline、marker、unavailable、测试 ID |
| 联动矩阵 | `doc/opt_strategy/evidence/stage_20/timeline-map-focus.md` | day_id/segment_id、触发、预期、人工结果 |
| 导出技术记录 | `doc/opt_strategy/evidence/stage_20/export-map-capture.md` | html2canvas 结论、SVG 快照方案、Stage 25 输入 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_20/manual-check.md`、`rollback-check.md` | M01～M05、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 18～20 跨 Stage 一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 10B、14、16、17、19 实际 Done，并在开工前复核 RouteSegment、TripPlan 正式 schema、时间轴 focus、地图 Key 配置和导出捕获边界。当前不表示 Ready、Done、Gate C、设计冻结或允许修改工程代码。
