# Stage 16：分级 Fallback、结果状态与新链路切换

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 16 |
| 目标 | 删除所有虚构事实 fallback，按 `complete/partial/failed × degraded` 返回唯一正式 `TripPlan`，并在 Gate B 通过后将 TripCoordinator 新链路与 Vue 新 schema 切为默认 |
| 对应优化项 | P1-1 分级 Fallback 与结果状态；核心规划闭环到默认入口的迁移 |
| 前置依赖 | Stage 15（并复核 Stage 13D/13E/14 的状态、Store 与展示契约） |
| 后续阻断 | 未通过时不得开始 Stage 17、18、20、21、23、28、31A；不得宣告 Gate B 可用 |
| 默认流量影响 | Gate B 证据齐备后，`TRIP_PLANNER_MODE` 默认值切为 `new`；legacy 只作短期受控回滚，shadow 仍不在普通请求双跑 |

## 2. 当前代码基线与差异

- 旧 `trip_planner_agent.py` 与路由仍可能使用北京默认坐标、算法平移、空值/虚构名称或 LLM 补事实，并将异常统一为看似成功或 500；这些路径必须连同回滚入口一并清除。
- 13D 已定义可信 complete/partial/failed 和首日失败边界，15 已建立技术错误 envelope，14/13E 已预留只读 status 展示/Store；但普通 `/trip/plan` 仍是 legacy 成功 wrapper，`new` 仍未就绪。
- 真实 cache/stale 由 17 才提供；16 只消费来源 freshness，默认 `degraded=false`，不得凭空制造 stale 或把技术失败降级为 plan。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改/删除 | `backend/app/agents/trip_planner_agent.py` 及 legacy fallback 辅助函数 | 删除 `_create_fallback_plan`、北京默认坐标、平移、虚构 POI/酒店/路线/费用和 LLM 补事实分支 | 静态扫描与注入测试均无可达虚构路径 |
| 修改 | `backend/app/models/trip.py`、`models/__init__.py` | 固定正式 `TripPlan` 及 complete/partial/failed/degraded/first_failed_date/missing/accepted 约束 JSON | failed 无 days，partial 只连续前缀 |
| 修改 | `backend/app/coordinators/trip_coordinator.py`、`services/trip_assembler.py` | 只从 Validator.valid 可信输入组装 complete/partial；技术错误走 15 | 不以 fallback 绕过验证 |
| 修改 | `backend/app/config.py`、`backend/.env.example` | 将 `TRIP_PLANNER_MODE` 默认改为 `new`，保留严格枚举和短期回滚说明 | mode/默认/启动日志可验证且不含 secret |
| 修改 | `backend/app/api/routes/trip.py`、`api/error_mapping.py` | `/trip/plan` 在 new 返回正式 TripPlan 或 15 的 error envelope；legacy/shadow 受控 | 普通 new 不访问旧 Planner |
| 修改 | `frontend/src/types/index.ts`、`services/api.ts` | 原位替换 legacy success 类型/解码为正式 `TripPlan`，升级 13E snapshot version/guard | 不保留长期 adapter 或旧 response 类型 |
| 修改 | `frontend/src/stores/current_trip.ts`、`views/Home.vue`、`views/Result.vue`、`components/TripStatusBanner.vue` | 以新 TripPlan 原子替换，正确显示 complete/partial/failed/degraded | failed 不当作可编辑/完整行程，partial 不丢首个失败日 |
| 新增 | `backend/tests/api/test_trip_new_mode.py`、`backend/tests/integration/test_no_fabricated_fallback.py` | 覆盖 new 默认、mode、真实性、status 与 rollback | 全部 fake、离线 |
| 新增 | `doc/opt_strategy/evidence/stage_16/` | 保存 Gate B 输入、模式矩阵、schema/真实性、前端与回滚记录 | 不提交完整 plan/payload |

## 4. 输入、输出与迁移契约

### 4.1 唯一正式结果

`new` 模式的 `/api/trip/plan` 成功响应体就是正式 `TripPlan`，不得再包装为 legacy `{success,message,data}`：

| `plan_status` | 响应与内容 |
|---|---|
| `complete` | HTTP 200；N 个连续 valid days、N 条 lodging、完整路线/timeline/budget/accepted constraints；`degraded` 独立布尔值 |
| `partial` | HTTP 200；仅 Day1..DayK（1≤K<N）及 K+1 lodging、first_failed_date、截断/missing 说明；DayK 仍按原请求非末日验证 |
| `failed` | HTTP 200；request_id、稳定业务 reason、失败摘要、accepted constraints；不携带 `days` 字段、路线、预算或酒店，也不得伪装 partial 数据 |
| 技术错误 | Stage 15 HTTP error envelope；不包含 TripPlan |

- 前端/API 同一提交同步切换到上述 schema；删除 legacy `TripPlanResponse` 成功类型与 adapter。无其他调用方，不维持 API 版本或长期兼容层。
- 13E 将 snapshot 从 `v1` 升至与正式 schema 对应的 `v2`；仅当旧 v1 能经确定、完整的字段映射验证时迁移，否则清除其 Store key 并显示无当前行程，不猜测转换。failed 也可作为正式状态快照，但不保存技术 error/token/raw diagnostics。
- `degraded=true` 仅可从来源 metadata 的可信 stale 输入传入，和 complete/partial/failed 正交；Stage 17 前默认为 false。前端必须显示时效提示而不改变 plan_status。

### 4.2 真实性与禁止 fallback

- 坐标只可来自本次 AmapService 地理编码/POI 或带来源 metadata 的可用缓存；坐标缺失/非法、POI/route ID 未知、必要路线缺失或 Agent 非法 ID 必须进入既定 repair、业务不可行或技术错误，禁止北京默认值、字符串拼坐标、算法平移或直线。
- 名称、地址、酒店、餐饮、路线距离/耗时/polyline、票价/预算项均只能来自 sourced/estimated 责任方；LLM 只能返回允许的选择/建议字段，绝不能补写事实。
- accepted constraint 不满足、required POI 不唯一/未安排、food 下限不满足均为 `failed`，不得删相关日期生成 partial；首日不能完整形成同样 failed。
- 回滚到 legacy 不得恢复已删除的虚构 fallback；它只能使用已清除虚构分支的旧实现，且记录为短期运行模式，不得继续堆叠新能力。

### 4.3 Mode 与切换顺序

1. Gate B 所有 Stage 5～13D 的实际 Done、Stage 13E/14 前端 schema 证据、Stage 15 error 证据与本 Stage 离线/人工证据均通过后，才允许把默认 `TRIP_PLANNER_MODE` 设为 `new`。
2. `new`：API 在线程池执行 Coordinator；返回正式 TripPlan 或 error envelope；绝不调用旧 Planner/fallback。
3. `legacy`：仅紧急短期回滚使用；成功响应可保留旧 shape，但实现也不得含虚构 fallback。`shadow`：只允许显式本地验证，不在普通用户请求双跑/额外消耗外部调用；失败只记录脱敏差异。
4. 回滚演练从 new→legacy→new，分别验证入口、无旧虚构分支、session snapshot 处理和错误/status；Stage 33 前复核并删除 legacy/shadow 开关与实现。

## 5. 配置与功能开关

| 项目 | 引入前 | Stage 16 后 | 回滚 | 清理 |
|---|---|---|---|---|
| `TRIP_PLANNER_MODE` | legacy 默认，new reserved | new 默认；合法值 legacy/shadow/new | 显式 legacy，禁止虚构 fallback | Stage 33 复核/收敛 |
| 前端 snapshot | v1 legacy carrier | v2 正式 TripPlan | 仅确定性迁移/清除本 Store key | Stage 33 清迁移分支 |

不新增其他环境变量、缓存或 Provider 自动切换。

## 6. 实施步骤

1. 列出并删除所有旧 Planner/路由/辅助函数中的虚构事实 fallback，先以静态与 fake 注入测试锁定禁止行为。
2. 收口正式 TripPlan status/source/failure schema，保证 Coordinator/Assembler 只接受可信验证输入。
3. 让 API dispatcher 按 mode 执行 new Coordinator、15 error mapping；将 new 设为默认的代码变更保持 Gate B 检查点。
4. 同提交切换前端类型、API decoder、Store snapshot v2 和 14 status 展示，移除 legacy success adapter。
5. 完成 new/legacy/shadow mode、complete/partial/failed/degraded、非法输入和 rollback 演练；Gate B 汇总后启用默认 new。

## 7. 明确不做

- 不引入缓存/stale 产生机制、并发、SSE、天气重排、地图、PDF、历史、对话或生产部署。
- 不放宽 accepted constraints、修复上限、partial 前缀/DayK/K+1 lodging 规则，也不在前端改写事实。
- 不保留长期 API v1/v2、legacy/new response adapter 或普通流量 shadow 双跑。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S16-T01 | unit | 坐标/POI/route/价格缺失或非法 | 无北京/平移/直线/LLM fallback，进入既定错误/业务路径 |
| S16-T02 | integration | complete/partial/failed | 13D 语义、days、DayK/K+1、accepted 边界不漂移 |
| S16-T03 | api | new 默认 success/error schema | TripPlan 直出或 15 envelope，无 legacy wrapper |
| S16-T04 | api | mode 矩阵 | new 不调旧 Planner；shadow 普通请求零双跑；legacy 仅受控 |
| S16-T05 | integration | required/accepted/首日失败 | failed，不生成 partial/fallback |
| S16-T06 | contract | frontend v2 schema/snapshot | complete/partial/failed/degraded guard 与原子替换正确 |
| S16-T07 | regression | 旧 fallback 静态/运行时扫描 | 禁止符号/默认坐标/虚构事实均不可达 |
| S16-T08 | regression | Gate B 上游与 build | Stage 1～15 关键回归、前端 build 通过 |

```text
python -m pytest backend/tests/api/test_trip_new_mode.py backend/tests/integration/test_no_fabricated_fallback.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S16-M01 | 默认 new 生成 complete/partial/failed fixtures | 页面与网络分别按正式 TripPlan/status 呈现，无 legacy wrapper |
| S16-M02 | 注入空 POI、坐标/路线失败、Agent 非法 ID | 不显示北京、直线、虚构名称/价格或成功假计划 |
| S16-M03 | 执行 new→legacy→new 回滚演练 | 每次入口可核验，未恢复虚构 fallback，当前 Store 不被旧/迟到响应覆盖 |

## 10. 量化退出指标

- [ ] S16-T01～T08、S16-M01～M03 通过，三条命令退出码为 0。
- [ ] 默认 new 请求调用 legacy Planner/fallback、普通 shadow 双跑、虚构坐标/实体/路线/费用的次数均为 0。
- [ ] partial 含跳日/accepted 缺失/错误 DayK 语义，failed 含 days 或技术错误伪装业务状态的次数均为 0。
- [ ] 前端 legacy success adapter/双 schema 字段存活数为 0；v2 snapshot 错误迁移或敏感持久化数为 0。
- [ ] Gate B 所需 Stage 证据缺失时将 default 切 new 的次数为 0；切换/回滚后错误 request_id 与前端状态覆盖异常数为 0。

## 11. 迁移与回滚

- 迁移按“删除虚构分支→正式 schema→API mode→前端 v2→Gate B/default new”顺序；不得先切流量再补真实性/前端契约。
- 发现回归时先显式设为 legacy，保留新 schema/证据以定位；只有下游依赖逆序回滚后才可撤除 new 路径。legacy 回滚实现仍不得恢复任何虚构 fallback。
- 回滚后运行 S16-T01～T05、build、M02～M03，并记录 mode 值、入口调用计数、snapshot 处置和结果。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_16/test-summary.md` | 命令、8 用例、离线、失败数 |
| 真实性矩阵 | `doc/opt_strategy/evidence/stage_16/no-fabrication-matrix.md` | 旧分支、替代行为、测试 ID |
| Mode/schema 矩阵 | `doc/opt_strategy/evidence/stage_16/mode-schema-matrix.md` | mode、入口、响应、snapshot、调用数 |
| Gate B 汇总输入 | `doc/opt_strategy/evidence/stage_16/gate-b-input.md` | 必需 Stage 证据、缺口与切换结论 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_16/manual-check.md`、`rollback-check.md` | M01～M03、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化，待本批 Stage 14～16 交接审计及 Stage 9A～16 全范围一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 15 实际 Done，并在切换前复核 Gate B、无虚构 fallback、正式 schema、mode 默认值、13E snapshot v2 与 new→legacy→new 回滚。当前不表示 Ready、Done、设计冻结或允许修改工程代码。
