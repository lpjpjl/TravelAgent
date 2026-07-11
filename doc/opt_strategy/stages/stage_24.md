# Stage 24：图片加载优化

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 24 |
| 目标 | 让结果页图片在全部展开和导出准备场景下具有可判断的加载终态，保证成功或最终 fallback 后 `imagesReady` 可被 Stage 25 等待 |
| 对应优化项 | P2-5 图片加载优化 |
| 前置依赖 | Stage 22、23；开工前复核多天全部展开能力、图片批量接口、Result 图片组件、导出容器复制方式和 html2canvas 图片限制 |
| 后续阻断 | 未通过时不得开始 Stage 25 PDF 导出重构 |
| 默认流量影响 | 改善图片显示、占位、重试和 ready 状态；不改变图片批量接口 contract、TripPlan、PDF 生成或后端 Provider 语义 |

## 2. 当前代码基线与差异

- 当前 Result 图片通过 `<img :src="getAttractionImage(...)" @error="handleImageError">` 加载，失败后直接替换为内联 SVG fallback，但没有 loading/success/error/fallback 状态机。
- 当前图片容器有固定样式，但没有可复用骨架屏、懒加载策略、有限重试次数或可被导出流程等待的 `imagesReady`。
- 当前 PDF/图片导出复制 DOM 后直接 html2canvas，无法判断图片是否已经成功加载或进入最终 fallback，存在慢网/404/超时导致导出不稳定的风险。
- Stage 23 负责批量返回图片 URL 和单项状态；Stage 24 只消费这些结果，不改变批量接口语义。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 新增 | `frontend/src/components/AttractionImage.vue` 或等价组件 | 封装图片显示、骨架、懒加载、重试、fallback 和事件 | Result 不再散落图片状态逻辑 |
| 新增/修改 | `frontend/src/composables/useTripImages.ts` | 管理批量图片结果、单项状态机、`imagesReady` 和导出等待 promise | Stage 25 可复用 |
| 修改 | `frontend/src/views/Result.vue` | 接入图片组件和 ready 状态；全部展开时能触发可见图片加载 | 1～3 天全部展开后图片均达终态 |
| 修改 | `frontend/src/types/index.ts` | 定义图片加载状态、fallback reason 和 ready summary 类型 | 前后端状态边界清晰 |
| 新增 | `doc/opt_strategy/evidence/stage_24/` | 保存慢网、404、超时、重试耗尽、导出 ready 和回滚证据 | 不含真实 Key 或完整图片 URL 列表 |

若当前前端没有测试框架，本 Stage 采用构建、静态扫描、浏览器 Network/DevTools 人工验收和记录化证据；不强制引入新测试框架。

## 4. 图片状态契约

### 4.1 单项状态机

| 状态 | 进入条件 | 终态 |
|---|---|---|
| `idle` | 图片项尚未进入可加载范围 | 否 |
| `queued` | 已有批量结果，等待可见或导出预加载 | 否 |
| `loading` | 浏览器正在加载图片 URL | 否 |
| `loaded` | 图片 onload 成功 | 是 |
| `retrying` | onerror/timeout 后仍有剩余重试 | 否 |
| `fallback` | not_found、重试耗尽、无 URL 或超时终止 | 是 |

- `failed` 不作为无限等待状态；对 UI 和导出而言，最终必须转为 `fallback`。
- fallback 必须是轻量、同尺寸、可被 html2canvas 捕获的本地占位，不依赖外部网络。
- 状态 key 使用 Stage 23 的 `client_ref`，不得只用景点名称。

### 4.2 `imagesReady`

`imagesReady` 表示当前 TripPlan 中需要导出的图片全部进入终态：

```text
imagesReady = all(image.status in ["loaded", "fallback"] for visible/export-required images)
```

| 场景 | 要求 |
|---|---|
| 普通浏览 | 可懒加载；未展开天的图片可保持 idle/queued |
| 全部展开 | 所有展开天级卡片图片必须开始加载并最终进入 loaded/fallback |
| Stage 25 导出准备 | 导出容器或导出模式必须触发全部导出图片加载，并等待 loaded/fallback |
| TripPlan 替换 | 清理旧图片状态和 pending timer，重新计算 ready |

- `imagesReady` 必须有超时保护；超时项进入 fallback，不得让导出流程无限等待。
- ready summary 至少包含 total、loaded、fallback、pending、failed/retryExhausted。
- `imagesReady=true` 不代表图片来源真实可用，只代表 UI 已进入可导出终态。

### 4.3 懒加载、尺寸与重试

| 项 | 约束 |
|---|---|
| 懒加载 | 普通浏览可使用 native `loading="lazy"` 或 IntersectionObserver |
| 导出模式 | 必须绕过懒加载等待盲区，主动触发导出所需图片 |
| 尺寸 | 图片容器固定宽高或 aspect-ratio，骨架、图片、fallback 尺寸一致 |
| 重试 | 有限次数，默认 1～2 次；每次有短间隔和总超时 |
| fallback | 本地 SVG/data URL 或静态资源，占位语义清晰 |
| CORS | 对可能进入 html2canvas 的跨域图片记录 `crossOrigin` 策略和失败降级 |

## 5. 配置与功能开关

不新增用户功能开关。

| 配置/常量 | 默认 | 说明 |
|---|---|---|
| `IMAGE_RETRY_LIMIT` | 1 或 2 | 单项有限重试次数 |
| `IMAGE_LOAD_TIMEOUT_MS` | 8000～12000 | 单项加载超时上限 |
| `IMAGES_READY_TIMEOUT_MS` | 根据 1～3 天最大图片数确定 | 导出准备整体等待上限 |

若实现为常量，需在证据中记录；若实现为 env，需登记到 04 配置表并更新 `.env.example`。

## 6. 实施步骤

1. 基于 Stage 23 批量结果定义前端图片状态类型和 `client_ref` 映射。
2. 抽出图片组件，统一骨架、真实图片、fallback、错误和重试显示。
3. 实现 `useTripImages` 或等价 composable，提供状态机、summary、`imagesReady` 和 `waitForImagesReady()`。
4. 接入 Result：普通浏览懒加载，全部展开时触发对应图片加载，TripPlan 替换时清理旧状态。
5. 为 Stage 25 记录导出准备契约：如何触发全部图片加载、等待终态、处理超时/fallback。
6. 完成慢网、404、超时、重试耗尽、全部展开和回滚证据。

## 7. 明确不做

- 不新增图片批量接口、不改变 Stage 23 请求/响应 contract。
- 不实现 PDF 分页、导出 Blob、历史保存、地图快照或自动下载。
- 不改变 TripPlan 图片字段真实性，不把 fallback 写回后端或 Store 事实字段。
- 不做移动端布局重构，不引入新图片 CDN 或图片代理。
- 不以无限 spinner、空白图片或静默失败作为成功状态。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S24-T01 | unit/static | 状态机成功加载 | idle/queued/loading -> loaded，ready summary 正确 |
| S24-T02 | unit/static | 404 或 onerror | 有限重试后进入 fallback |
| S24-T03 | unit/static | 超时 | 超时项进入 fallback，不无限 pending |
| S24-T04 | unit/static | TripPlan 替换 | 清理旧状态、timer 和 ready promise |
| S24-T05 | integration/static | 全部展开 | 触发所有展开天图片加载并最终 ready |
| S24-T06 | regression | Stage 23 单项 failed/not_found | 被消费为 fallback，不导致整页崩溃 |
| S24-T07 | build | 前端构建 | build 通过，无类型错误 |

```text
npm --prefix frontend run build
rg -n "imagesReady|waitForImagesReady|retry|fallback|loading=\"lazy\"|IntersectionObserver|attraction-image" frontend/src
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S24-M01 | 普通网络打开结果页 | 可见图片逐步加载，布局不跳动 |
| S24-M02 | 全部展开 1～3 天 | 全部图片进入 loaded 或 fallback，summary 无 pending |
| S24-M03 | DevTools 模拟慢网 | 有骨架/占位，最终不无限等待 |
| S24-M04 | 拦截某些图片 404 | 对应项进入 fallback，其他图片正常 |
| S24-M05 | 模拟图片 URL 跨域/canvas 失败 | 记录降级，Stage 25 可等待终态 |

## 10. 量化退出指标

- [ ] S24-T01～T07、S24-M01～M05 通过，命令退出码为 0。
- [ ] 全部展开后 pending 图片数量最终为 0。
- [ ] 慢网、404、超时、重试耗尽导致导出等待无限挂起次数为 0。
- [ ] 图片容器加载前后布局位移和高度跳变导致关键内容遮挡次数为 0。
- [ ] fallback 被写回 TripPlan 事实字段或后端持久化的次数为 0。

## 11. 迁移与回滚

- 迁移顺序：类型/状态机 → 图片组件 → Result 接入 → 全部展开触发 → Stage 25 ready 契约记录。
- 回滚时先恢复 Result 的图片渲染，再移除 composable；不得留下永远 pending 的 `imagesReady`。
- 回滚后运行 S24-T02～T07、前端 build 和 M02～M04。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_24/test-summary.md` | build、扫描、T01～T07 结果 |
| 状态机矩阵 | `doc/opt_strategy/evidence/stage_24/image-state-machine.md` | 状态、事件、终态、fallback reason |
| ready 契约 | `doc/opt_strategy/evidence/stage_24/images-ready-contract.md` | summary、wait API、超时和 Stage 25 消费方式 |
| 慢网/失败记录 | `doc/opt_strategy/evidence/stage_24/failure-scenarios.md` | 慢网、404、超时、重试耗尽、跨域降级 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_24/manual-check.md`、`rollback-check.md` | M01～M05、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 22～24 跨 Stage 一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 22 和 Stage 23 实际 Done，并在开工前复核全部展开、图片批量 contract、导出容器复制方式和 html2canvas 图片限制。当前不表示 Ready、Done、Gate D、设计冻结或允许修改工程代码。
