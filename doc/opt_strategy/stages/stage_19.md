# Stage 19：本地地图 Key 风险治理

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 19 |
| 目标 | 在地图增强前完成本地开发高德 Web JS Key 的来源限制、前后端 Key 隔离、配置检查和脱敏证据，降低公开 Key 被滥用风险 |
| 对应优化项 | P1-6 地图 Key 风险治理，并为 Gate C 的前后端 Key 隔离提供设计证据 |
| 前置依赖 | Stage 1；开工前复核当前前端地图加载方式、`VITE_AMAP_WEB_JS_KEY`、后端 `AMAP_API_KEY` 和 `.env.example` |
| 后续阻断 | 未通过时不得开始 Stage 20 地图真实路线联动；不得宣告 Gate C |
| 默认流量影响 | 不改变规划 API 和 TripPlan；只影响本地地图能力是否可启用、配置错误提示和人工控制台治理 |

## 2. 当前代码基线与差异

- 当前 `frontend/src/views/Result.vue` 直接读取 `import.meta.env.VITE_AMAP_WEB_JS_KEY` 加载高德 Web JS API，但缺少独立配置校验、来源白名单说明和构建产物 Key 泄漏检查。
- 后端 `AMAP_API_KEY` 用于服务端 Provider/MCP/HTTP 能力；前端 Web JS Key 是浏览器公开 Key，两者不能混用。
- 当前 `.env.example`/启动检查尚未系统说明本地来源限制、配额、最小权限和未来正式域名变更要求。
- Stage 20 需要可靠加载前端地图；因此 Key 风险治理必须先完成，避免地图增强时继续扩大公开 Key 暴露面。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `frontend/src/config/amap.ts` 或等价配置模块 | 集中读取 `VITE_AMAP_WEB_JS_KEY`，导出地图能力状态和脱敏诊断 | 组件不直接散落读取 env |
| 修改 | `frontend/src/views/Result.vue` | 使用集中配置加载地图；缺失 Web JS Key 时显示地图不可用，不尝试后端 Key | 无 Key 时页面可用且不报敏感信息 |
| 修改 | `frontend/.env.example` | 增加 `VITE_AMAP_WEB_JS_KEY` 占位、用途、来源限制说明 | 不含真实 Key |
| 修改 | `backend/.env.example`、`backend/app/config.py` | 明确 `AMAP_API_KEY` 仅服务端使用；后端不向前端暴露服务 Key | 启动/日志只显示已配置/未配置 |
| 新增 | `doc/opt_strategy/evidence/stage_19/amap-console-checklist.md` | 记录高德控制台人工配置检查项 | localhost/127.0.0.1、配额、权限、无通配符 |
| 新增 | `frontend/src/__tests__` 或现有测试位置 | 覆盖前端配置解析和缺失 Key 行为 | 不引入真实 Key |
| 新增 | `backend/tests/unit/test_config_security.py` | 覆盖后端 Key 不进入公开配置/日志 | 离线 |
| 新增 | `doc/opt_strategy/evidence/stage_19/` | 记录配置、构建扫描、人工验收和回滚证据 | 脱敏 |

如当前项目没有前端单元测试框架，本 Stage 只记录类型检查/build 与人工 Network/构建产物扫描，不强制引入新框架。

## 4. 配置与安全契约

### 4.1 Key 边界

| Key | 所属环境 | 用途 | 可见性 | 禁止事项 |
|---|---|---|---|---|
| `AMAP_API_KEY` | 后端 | AmapService Provider/MCP/HTTP | secret，仅服务端 | 禁止进入 `VITE_`、前端 bundle、HTML、Network 响应或日志 |
| `VITE_AMAP_WEB_JS_KEY` | 前端 | 高德 Web JS 地图加载 | 浏览器公开，但受来源/配额限制 | 禁止复用后端服务 Key；禁止任意来源通配符 |

- 浏览器公开 Key 不等于可随意泄露；必须通过高德控制台来源限制、配额和最小权限降低风险。
- 未来加入正式域名时，必须显式更新白名单和证据，不预留 `*`、任意二级域名或无法追踪的来源。

### 4.2 本地来源白名单

最小允许来源：

```text
http://localhost:5173
http://127.0.0.1:5173
http://localhost:3000
http://127.0.0.1:3000
```

- 端口以实际本地开发服务器为准；未使用端口不必加入。
- 不允许 `file://`、公网 IP、任意来源通配符或未说明的内网网段。
- 控制台截图/记录必须脱敏，只记录配置项、日期和结论，不提交完整 Key。

### 4.3 缺失与错误行为

- 缺失 `VITE_AMAP_WEB_JS_KEY`：前端隐藏或降级地图区域，显示稳定“地图配置不可用”状态；规划结果、时间轴、预算和导出非地图部分不受影响。
- 缺失 `AMAP_API_KEY`：后端按 Stage 1/15 的配置错误处理，不由前端 Web JS Key 兜底。
- Key 非法或来源被拒：地图加载失败只影响地图 UI，不回写 TripPlan，不触发重新规划，不暴露 Key 或完整错误对象。

## 5. 配置与功能开关

沿用 04 已登记配置：

| 配置 | 类型/敏感性 | 首次使用 Stage | 缺失行为 |
|---|---|---|---|
| `VITE_AMAP_WEB_JS_KEY` | 浏览器公开 Key | 19 | 地图能力关闭或明确报配置错误，不回退后端 Key |
| `AMAP_API_KEY` | secret | 1 | 服务端高德能力不可用，按配置错误处理 |

不新增地图开关。Stage 20 地图组件根据 `VITE_AMAP_WEB_JS_KEY` 和加载状态自然降级。

## 6. 实施步骤

1. 建立前端地图配置读取边界和脱敏诊断，替换 Result 中直接 env 读取。
2. 更新前后端 `.env.example` 和本地启动说明，明确两类 Key 的用途、可见性和禁止混用。
3. 增加构建产物/源码扫描命令，验证 `AMAP_API_KEY` 不进入前端 bundle，证据中不出现真实 Key。
4. 完成人工控制台配置检查：localhost/127.0.0.1 来源、实际端口、配额、最小权限、无通配符。
5. 验证缺失/非法 Web JS Key 时页面稳定降级，地图错误不影响 TripPlan 状态。

## 7. 明确不做

- 不申请、生成、轮换或提交任何真实 Key。
- 不实现正式生产域名、CDN、防盗链代理、后端地图瓦片代理或地图功能增强。
- 不改变 AmapService Provider、路线、天气、缓存、规划 API 或 TripPlan schema。
- 不把前端 Web JS Key 当作 secret 存入后端，也不把后端服务 Key 暴露给浏览器。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S19-T01 | frontend | `VITE_AMAP_WEB_JS_KEY` 缺失 | 地图能力关闭，页面无崩溃，无后端 Key 兜底 |
| S19-T02 | frontend | 配置模块脱敏 | 只暴露 configured/disabled，不输出 Key |
| S19-T03 | backend | `AMAP_API_KEY` 日志/配置 | 只显示已配置/未配置，不打印值 |
| S19-T04 | security | 前端源码/构建扫描 | 未发现 `AMAP_API_KEY`、真实 Key 模式或后端 Key 占位泄漏 |
| S19-T05 | regression | 规划 API | Key 治理不改变 TripPlan/API schema |

```text
python -m pytest backend/tests/unit/test_config_security.py -q
npm --prefix frontend run build
rg -n "AMAP_API_KEY|真实Key占位|YOUR_REAL" frontend/dist frontend/src backend/app doc/opt_strategy/evidence/stage_19
```

若 `frontend/dist` 未生成或不纳入版本库，构建扫描在实施证据中记录实际构建输出目录。

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S19-M01 | 检查高德控制台 Web JS Key 来源限制 | 仅 localhost/127.0.0.1 和实际端口，无通配符 |
| S19-M02 | 检查配额和权限 | 权限最小化，配额合理，不使用后端服务 Key |
| S19-M03 | 缺失/错误 Web JS Key 打开结果页 | 地图明确不可用，其余行程可读 |
| S19-M04 | 浏览器 Network/Console 检查 | 不出现后端 Key、完整错误对象或敏感日志 |

## 10. 量化退出指标

- [ ] S19-T01～T05、S19-M01～M04 通过，命令退出码为 0。
- [ ] 前端 bundle、HTML、Network 响应、日志和证据中 `AMAP_API_KEY` 泄漏次数为 0。
- [ ] 控制台来源通配符、非本地来源、后端 Key 混用次数为 0。
- [ ] 缺失 Web JS Key 导致规划结果页崩溃、TripPlan 被清空或 API 重试的次数为 0。

## 11. 迁移与回滚

- 迁移先建立配置读取模块，再更新组件和文档，最后做控制台人工验收；不得先扩大地图功能再补 Key 治理。
- 回滚时恢复 Result 原地图加载入口，但仍保留 `.env.example` 中前后端 Key 分离说明和禁止真实 Key 的证据；不得回滚为后端 Key 暴露给前端。
- 回滚后运行 S19-T01～T04、前端 build 和 M03。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_19/test-summary.md` | 命令、5 用例、扫描范围、失败数 |
| 控制台清单 | `doc/opt_strategy/evidence/stage_19/amap-console-checklist.md` | 来源、端口、配额、权限、无通配符结论 |
| 构建扫描 | `doc/opt_strategy/evidence/stage_19/bundle-scan.md` | 构建目录、扫描模式、脱敏结果 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_19/manual-check.md`、`rollback-check.md` | M01～M04、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 18～20 跨 Stage 一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 1 实际 Done，并在开工前复核高德 Key 类型、当前 Result 地图加载、`.env.example`、前端构建扫描路径和控制台可配置项。当前不表示 Ready、Done、Gate C、设计冻结或允许修改工程代码。
