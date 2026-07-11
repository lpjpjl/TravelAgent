# Stage 25：导出重构、自动全展开与 PDF 下载

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 25 |
| 目标 | 在用户不预先展开天级卡片的情况下，自动准备完整导出视图，生成且只生成一个准确 PDF Blob，并触发浏览器下载 |
| 对应优化项 | P2-3 导出代码重构，并为 Stage 27 的“同一 PDF Blob 上传历史”提供前端输入 |
| 前置依赖 | Stage 20、22、24；开工前复核地图捕获/SVG 快照记录、全部展开 snapshot/restore、`imagesReady`/`waitForImagesReady()`、当前 html2canvas/jsPDF 基线 |
| 后续阻断 | 未通过时不得开始 Stage 26 固定目录 PDF 存储，也不得开始 Stage 27 双向导航与 PDF 历史入口 |
| 默认流量影响 | 改变导出流程和 PDF 下载；不改变 TripPlan、图片批量接口、地图路线事实、后端报告存储或历史入口 |

## 2. 当前代码基线与差异

- 当前 `Result.vue` 的图片导出和 PDF 导出各自复制 DOM、改写样式、截图，重复逻辑较多，失败恢复边界不清晰。
- 当前 PDF 导出依赖页面当前折叠状态；若天级卡片未展开，PDF 不包含完整天级内容。
- 当前 PDF 通过整张长截图按 A4 高度平移分页，可能把标题、时间轴事件、点位卡片、图片或餐饮信息切断。
- 当前地图导出直接抓高德 canvas；Stage 20 已要求若 html2canvas/地图 canvas 捕获不稳定，必须使用基于真实 RouteSegment polyline 的 SVG 路线快照，不能以空白或点位直线冒充成功。
- 当前 `pdf.save()` 直接保存，未显式产出可被 Stage 27 复用的单一 Blob，也没有“下载成功但上传留待后续”的边界。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 新增 | `frontend/src/services/export/exportPipeline.ts` 或等价模块 | 编排导出准备、截图、分页、Blob 生成、下载和恢复 | PDF 导出路径只有一个 pipeline |
| 新增 | `frontend/src/services/export/prepareExportView.ts` | 创建独立导出容器，自动全部展开，等待字体/图片/地图 | 不依赖页面当前折叠状态 |
| 新增/修改 | `frontend/src/services/export/paginatePdf.ts` | 卡片感知 A4 分页，避免关键块任意截断 | 1～3 天 PDF 跨页稳定 |
| 新增/修改 | `frontend/src/services/export/mapExport.ts` | 消费 Stage 20 html2canvas 记录或 SVG 真实路线快照 | 地图不空白、不画点位直线 |
| 修改 | `frontend/src/views/Result.vue` | 将 PDF 按钮接入统一 pipeline；图片导出可复用准备逻辑但不扩大本 Stage 成功标准 | 页面状态恢复，错误提示稳定 |
| 修改 | `frontend/src/composables/useDayExpansion.ts` | 确认 Stage 22 snapshot/expandAll/restore 可被导出容器或导出模式调用 | 导出结束/失败恢复原状态 |
| 修改 | `frontend/src/composables/useTripImages.ts` | 确认 Stage 24 `waitForImagesReady()` 支持导出模式 | 图片 loaded/fallback 后才截图 |
| 新增 | `doc/opt_strategy/evidence/stage_25/` | 保存 PDF 样例记录、分页矩阵、地图/图片等待、Blob 和回滚证据 | 不提交 PDF 文件本体、Key 或完整 TripPlan |

若当前前端没有测试框架，本 Stage 采用 `npm --prefix frontend run build`、静态扫描、记录化人工验收和必要的开发环境手工 PDF 抽查；不强制引入新测试框架。

## 4. 导出契约

### 4.1 用户行为

| 场景 | 行为 |
|---|---|
| 全部折叠状态点击 PDF 导出 | 导出流程自动准备全部天级内容，页面原状态最终恢复为全部折叠 |
| 部分展开状态点击 PDF 导出 | 导出流程自动准备全部天级内容，页面原状态最终恢复为原部分展开 |
| 全部展开状态点击 PDF 导出 | 导出流程保留完整内容，完成后仍全部展开 |
| 导出失败 | 清理导出容器、恢复原展开状态、提示稳定错误 |

- 导出按钮始终可点击，不要求用户预先展开任何天。
- 导出容器必须和用户当前页面状态隔离；不得在导出过程中永久改写 TripPlan、Store 或 day 展开事实。
- 结果状态为 failed 且无 days 时不得生成“成功 PDF”；应返回明确不可导出状态。

### 4.2 准备顺序

PDF pipeline 的顺序固定：

```text
snapshot expansion state
create isolated export container
expand all days in export context
wait for fonts
wait for imagesReady / waitForImagesReady()
prepare map snapshot or SVG real-route snapshot
layout and paginate
render to canvas/page images
create exactly one PDF Blob
trigger browser download
expose same Blob for Stage 27
cleanup and restore expansion state
```

- 任一步失败都必须进入 cleanup/finally，恢复用户原状态。
- 不允许在失败后静默导出缺少关键内容的 PDF 并提示成功。
- 只允许生成一个正式 PDF Blob；用于下载和 Stage 27 上传的必须是同一个 Blob 引用或同一份 Blob 数据。

### 4.3 地图导出

| 输入 | 要求 |
|---|---|
| Stage 20 html2canvas 可稳定捕获 | 可使用真实地图截图，必须包含酒店、点位和真实路线 |
| Stage 20 记录地图捕获不稳定 | 使用 Stage 20 `exportRouteSnapshot` 生成的真实 polyline SVG 快照 |
| 缺失路线 polyline | 显示明确“路线不可用”状态，不画端点直线 |
| 缺失 Web JS Key | 地图区域使用稳定降级，不影响非地图内容；若地图是本次成功标准的一部分，则记录不可用原因 |

- 地图不能重新请求路线、不能用景点坐标顺序连线、不能把空白地图当作成功。
- SVG 快照可无底图，但必须标明“无底图真实路线示意”，并仅使用 RouteSegment 真实 polyline。

### 4.4 图片与字体等待

- 字体必须通过 `document.fonts.ready` 或等价能力等待；不支持时记录降级。
- 图片必须消费 Stage 24 的 `waitForImagesReady()`；导出模式需要主动触发全部导出图片加载，绕过普通懒加载盲区。
- `loaded` 和 `fallback` 都是可导出终态；`idle`、`queued`、`loading`、`retrying` 不能截图。
- 图片等待超时必须由 Stage 24 转为 fallback；Stage 25 不创建第二套图片状态机。

### 4.5 A4 分页

分页必须以内容块为最小保护单位：

| 内容块 | 分页规则 |
|---|---|
| 页面标题/城市日期摘要 | 不拆分 |
| 天级 header | 不作为页尾孤儿标题 |
| 时间轴事件 | 单个事件尽量不拆；超高事件允许内部拆分并标记 |
| 景点卡片 | 卡片标题、图片和首段描述不任意分离 |
| 餐饮/住宿/预算小项 | 不在行内硬切 |
| 地图快照 | 不拆分；过高时按固定比例缩放 |

- 禁止仅用整页长图平移分页作为最终成功标准。
- 对超过单页高度的超长内容块，必须有明确降级策略：缩放、允许内部拆分或单独续页，并在证据中记录。

## 5. 配置与功能开关

不新增用户功能开关。

| 配置/常量 | 默认 | 说明 |
|---|---|---|
| `PDF_PAGE_FORMAT` | `a4` | 仅前端常量 |
| `PDF_RENDER_SCALE` | 2 或证据确定值 | 控制清晰度和性能 |
| `PDF_PREPARE_TIMEOUT_MS` | 继承图片/地图等待上限组合 | 导出准备总保护 |

本 Stage 不接入 `TRIP_REPORT_HISTORY_ENABLED`，不读取或写入后端 `TRIP_REPORT_DIR`。

## 6. 实施步骤

1. 抽取导出准备和截图公共逻辑，删除 PDF/图片导出中的重复 DOM 克隆与样式散落逻辑。
2. 接入 Stage 22 的展开状态 snapshot/restore 和导出上下文全部展开能力。
3. 接入 Stage 24 的 `waitForImagesReady()` 和字体等待，确保截图前所有图片进入 loaded/fallback。
4. 接入 Stage 20 地图捕获记录和真实路线 SVG 快照，拒绝空白地图或端点直线兜底。
5. 实现卡片感知分页和 PDF Blob 生成；下载使用同一 Blob，不直接依赖 `pdf.save()` 作为唯一出口。
6. 在全部折叠、部分展开、全部展开下验证 1/2/3 天导出，记录 PDF 抽查、Blob、回滚和失败恢复证据。

## 7. 明确不做

- 不实现后端 PDF 上传、固定目录存储、历史列表、历史下载或双向导航；这些属于 Stage 26～27。
- 不重新生成 TripPlan、不请求新路线、不改变地图 polyline、图片批量 contract、图片状态机或 day 展开事实。
- 不把 PDF Blob 存入 sessionStorage/localStorage，不把历史 PDF 作为可编辑行程恢复。
- 不做移动端导出适配，不引入服务端渲染、浏览器扩展或生产部署策略。
- 不用空白地图、点位直线、无限 spinner、截断关键卡片或多次生成 Blob 来冒充成功。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S25-T01 | unit/static | 导出准备顺序 | snapshot → expandAll → fonts/images/map → paginate → Blob → cleanup 顺序明确 |
| S25-T02 | unit/static | 失败恢复 | 任一步异常后清理导出容器并 restore 原展开状态 |
| S25-T03 | unit/static | 单一 Blob | 下载和 Stage 27 暂存使用同一个 Blob，不二次生成 |
| S25-T04 | unit/static | 地图输入 | 使用 Stage 20 捕获记录或真实 polyline SVG；缺 polyline 不画直线 |
| S25-T05 | unit/static | 图片等待 | 只在 loaded/fallback 后截图，pending 不可继续 |
| S25-T06 | unit/static | 分页规则 | 标题、天 header、点位卡片、地图不被任意硬切 |
| S25-T07 | build | 前端构建 | build 通过，无类型错误 |
| S25-T08 | static | 旧导出路径扫描 | PDF 逻辑不再保留独立重复 DOM 克隆/长图平移成功路径 |

```text
npm --prefix frontend run build
rg -n "exportAsPDF|html2canvas|jsPDF|pdf\\.save|output\\(|Blob|innerHTML|addImage|heightLeft|expandAllDays|waitForImagesReady|exportRouteSnapshot" frontend/src
```

扫描命令在实施时需结合最终代码人工判读：`html2canvas`、`jsPDF` 和 `addImage` 可以存在，但必须属于统一 pipeline 和分页策略。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S25-M01 | 全部折叠状态导出 1、2、3 天 PDF | PDF 包含全部天内容，导出后仍全部折叠 |
| S25-M02 | 部分展开状态导出 | PDF 完整，导出后恢复原展开集合 |
| S25-M03 | 全部展开状态导出 | PDF 完整，页面状态不被破坏 |
| S25-M04 | 注入慢图/404/fallback | PDF 等待 loaded/fallback，无无限等待 |
| S25-M05 | 地图 canvas 捕获失败 | 使用真实 polyline SVG 快照或明确失败，不导出空白地图 |
| S25-M06 | 抽查跨页边界 | 标题、时间轴事件、点位卡片、图片、餐饮信息无关键截断、重复或遗漏 |
| S25-M07 | 模拟导出中异常 | 清理隐藏容器，恢复状态，提示稳定错误 |

## 10. 量化退出指标

- [ ] S25-T01～T08、S25-M01～M07 通过，命令退出码为 0。
- [ ] 从全部折叠、部分展开、全部展开三类状态导出后，用户展开状态恢复正确率 100%。
- [ ] 1、2、3 天 PDF 中关键内容遗漏、重复、空白地图、点位直线假路线和关键卡片任意截断次数均为 0。
- [ ] 每次成功导出正式 PDF Blob 生成次数为 1。
- [ ] 导出失败后隐藏容器残留、Store/TripPlan 被修改、图片 pending 无限等待次数均为 0。

## 11. 迁移与回滚

- 迁移顺序：公共导出准备 → 状态 snapshot/restore → 图片/字体等待 → 地图快照 → 分页 → Blob 下载。
- 回滚时先恢复 PDF 按钮到旧路径，再移除新 pipeline；不得保留半接入的双路径导致生成两个 Blob 或状态不恢复。
- 回滚后运行 S25-T02～T05、前端 build、M01～M03，并确认 Stage 22/24 的独立功能仍可用。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_25/test-summary.md` | build、扫描、T01～T08 结果 |
| 导出流程记录 | `doc/opt_strategy/evidence/stage_25/export-pipeline.md` | 准备顺序、失败恢复、单 Blob 说明 |
| PDF 抽查矩阵 | `doc/opt_strategy/evidence/stage_25/pdf-sample-matrix.md` | 1/2/3 天、三种展开状态、内容抽查 |
| 分页记录 | `doc/opt_strategy/evidence/stage_25/pagination-check.md` | 关键块、跨页策略、异常块处理 |
| 地图/图片等待 | `doc/opt_strategy/evidence/stage_25/map-image-readiness.md` | Stage 20 快照输入、Stage 24 ready、失败场景 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_25/manual-check.md`、`rollback-check.md` | M01～M07、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 25 独立一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 20、22、24 实际 Done，并在开工前复核地图捕获/SVG 快照记录、全部展开 snapshot/restore、图片 ready contract 和当前导出代码基线。当前不表示 Ready、Done、Gate D、设计冻结或允许修改工程代码。
