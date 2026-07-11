# Stage 22：多天独立折叠与全部展开

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 22 |
| 目标 | 让 1～3 天天级卡片可以互不干扰地单独展开、全部展开和全部折叠，并为 Stage 25 的自动全展开导出提供稳定状态接口 |
| 对应优化项 | P2-7 天级独立折叠 |
| 前置依赖 | Stage 14；开工前复核 TripPlan day ID、时间轴 day 引用、Result 当前 `activeDays` 和 Ant Design Vue Collapse 行为 |
| 后续阻断 | 未通过时不得开始 Stage 24 的全部展开图片 ready 验证，也不得开始 Stage 25 PDF 导出重构 |
| 默认流量影响 | 只改变结果页天级卡片展开交互；不改变 TripPlan、时间轴事实、地图、图片 API、导出生成或后端接口 |

## 2. 当前代码基线与差异

- 当前 `frontend/src/views/Result.vue` 使用 `<a-collapse v-model:activeKey="activeDays" accordion>`，天然只允许同时展开一个面板，不满足 1～3 天全部展开。
- 当前面板 key 和 DOM id 基于数组下标 `index`，TripPlan 更新或 partial 前缀变化时容易和旧展开状态串扰。
- 当前默认展开第一天；目标要求默认全部折叠，并提供单独展开、全部展开、全部折叠。
- Stage 25 需要在独立导出容器中自动展开全部天级卡片，并在导出结束后恢复用户原有展开状态；Stage 22 只提供可复用状态接口，不实现 PDF 导出。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `frontend/src/views/Result.vue` 或拆出的 `TripDays.vue` | 移除 `accordion` 单展开限制，接入稳定 day key 的展开状态 | 1～3 天可同时展开 |
| 新增/修改 | `frontend/src/composables/useDayExpansion.ts` 或等价模块 | 管理 `expandedDayKeys`、单日切换、全部展开、全部折叠、TripPlan 变更重置 | 状态逻辑可测试、可供 Stage 25 复用 |
| 修改 | `frontend/src/types/index.ts` | 明确前端 day key 的派生规则或类型别名 | 不使用数组下标作为唯一事实来源 |
| 新增 | `doc/opt_strategy/evidence/stage_22/` | 保存构建、人工验收、状态矩阵和回滚证据 | 证据不含完整 TripPlan 或敏感输入 |

若当前前端没有测试框架，本 Stage 以 `npm --prefix frontend run build`、静态扫描和记录化人工验收为准；不强制引入新测试框架。

## 4. 展开状态契约

### 4.1 稳定 day key

展开状态必须使用稳定 day key：

```text
day_key = day.id || `${day.date}:${day.day_index}`
```

- 若 Stage 16 后正式 TripPlan 已提供 `day_id`，优先使用 `day_id`。
- 若当前代码仍只有 `date/day_index`，可临时使用派生 key，但必须在 evidence 中记录派生规则。
- DOM id、时间轴 focus、展开状态和导出容器引用必须使用同一 day key 派生，不再把数组下标当作稳定事实。

### 4.2 默认与操作

| 操作 | 行为 |
|---|---|
| 首次进入结果页 | 默认全部折叠 |
| 点击某一天 header | 只切换该 day key，不影响其他天 |
| 全部展开 | 当前 TripPlan 中全部有效 day key 进入展开集合 |
| 全部折叠 | 展开集合清空 |
| TripPlan 替换 | 清理不属于新 TripPlan 的旧 key，默认回到全部折叠 |

- partial 结果只操作实际返回的连续前缀 days。
- failed 无 days 时不渲染天级折叠控件。
- 编辑模式下的景点移动/删除不应改变 day 展开状态，除非当前 TripPlan 被整体替换。

### 4.3 Stage 25 可复用接口

Stage 22 必须提供一个不依赖按钮点击的展开控制能力：

| 接口能力 | 用途 |
|---|---|
| `expandAllDays()` | Stage 25 导出容器自动展开全部天 |
| `collapseAllDays()` | 用户操作和导出失败恢复 |
| `snapshotExpansionState()` | Stage 25 保存用户原始状态 |
| `restoreExpansionState(snapshot)` | Stage 25 导出结束或失败后恢复 |

本 Stage 不创建导出容器、不等待图片、不生成 PDF。

## 5. 配置与功能开关

不新增配置和功能开关。

## 6. 实施步骤

1. 梳理 Result 中天级卡片 key、DOM id、滚动定位、时间轴引用和编辑交互。
2. 建立稳定 day key 派生函数和展开状态管理模块。
3. 移除 Collapse 的 `accordion` 单展开限制，默认全部折叠。
4. 增加全部展开/全部折叠入口，并保证 TripPlan 替换后清理旧状态。
5. 暴露 Stage 25 可复用的 snapshot/restore 能力，但不实现导出逻辑。
6. 执行构建、扫描、1/2/3 天人工验收和回滚记录。

## 7. 明确不做

- 不实现 PDF 导出重构、图片 ready、图片批量接口、SSE、历史入口或移动端适配。
- 不修改后端 TripPlan schema、时间轴计算、地图 polyline、预算、校验或规划链路。
- 不把展开状态写入后端，不把历史 PDF 作为可编辑 TripPlan 恢复。
- 不改变页面整体布局，只做天级折叠交互和必要的控制入口。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S22-T01 | unit/static | day key 派生 | 同一 TripPlan 下 key 稳定，TripPlan 替换后旧 key 被清理 |
| S22-T02 | unit | 单日切换 | 展开/折叠某一天不影响其他天 |
| S22-T03 | unit | 全部展开/全部折叠 | 1、2、3 天均能全部展开，再全部折叠 |
| S22-T04 | regression | partial/failed | partial 只操作返回 days；failed 无 days 不渲染折叠 |
| S22-T05 | static | Collapse 配置 | 不再使用 `accordion` 限制多天展开 |
| S22-T06 | build | 前端构建 | build 通过，无类型错误 |

```text
npm --prefix frontend run build
rg -n "accordion|activeDays|expandedDay|collapseAll|expandAll" frontend/src
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S22-M01 | 打开 1 天行程 | 默认折叠，可展开和折叠 |
| S22-M02 | 打开 2 天行程，依次展开两天 | 两天可同时展开，互不影响 |
| S22-M03 | 打开 3 天行程，点击全部展开/全部折叠 | 全部天级卡片状态正确 |
| S22-M04 | 生成新 TripPlan 后返回结果页 | 旧展开状态不串扰 |
| S22-M05 | 编辑景点顺序或删除景点 | 当前 day 展开状态不被意外重置 |

## 10. 量化退出指标

- [ ] S22-T01～T06、S22-M01～M05 通过，命令退出码为 0。
- [ ] 1、2、3 天行程全部展开成功率为 100%。
- [ ] 展开/折叠任一天导致其他天状态变化的次数为 0。
- [ ] TripPlan 替换后旧 day key 残留或错展次数为 0。
- [ ] 因本 Stage 修改导致 TripPlan、地图、图片 API 或 PDF 导出行为变化的次数为 0。

## 11. 迁移与回滚

- 迁移顺序：day key 工具 → 展开状态模块 → Result 接入 → 全部展开/折叠入口 → Stage 25 预留接口。
- 回滚时先恢复 Result 调用，再移除状态模块；不得回滚为只能单天展开并误标 Stage 25 可用。
- 回滚后运行 S22-T02～T06、前端 build 和 M01～M03。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_22/test-summary.md` | build、扫描、T01～T06 结果 |
| 展开矩阵 | `doc/opt_strategy/evidence/stage_22/expansion-matrix.md` | 1/2/3 天、单日、全部展开/折叠、TripPlan 替换 |
| Stage 25 接口记录 | `doc/opt_strategy/evidence/stage_22/export-expansion-contract.md` | snapshot/restore/expandAll/collapseAll 能力 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_22/manual-check.md`、`rollback-check.md` | M01～M05、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 22～24 跨 Stage 一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 14 实际 Done，并在开工前复核 TripPlan day ID、时间轴 day 引用和 Result 当前折叠行为。当前不表示 Ready、Done、Gate D、设计冻结或允许修改工程代码。
