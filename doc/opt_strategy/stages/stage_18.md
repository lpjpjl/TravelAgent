# Stage 18：天气驱动两次草案规划

> 权威总纲：[04_stage_implementation_spec.md](../04_stage_implementation_spec.md)
>
> 本文件是 04 实施规格的组成部分；约束优先级、Ready/Done、测试、配置、证据和回滚通用规则以总纲为准。

## 1. 工作包元数据

| 项目 | 内容 |
|---|---|
| Stage | 18 |
| 目标 | 将 Stage 4 的结构化天气从展示信息升级为规划输入，在真实预报范围内影响主要点位筛选/替换与 PrimaryDraft 分天排序 |
| 对应优化项 | P1-5 天气驱动决策，并为 Gate C 的天气范围、真实性和重建链路提供设计证据 |
| 前置依赖 | Stage 4、13D、16、17；开工前复核 Stage 7A/7B/8 的天气零影响预留、Stage 13D 下游重建、Stage 15 错误分流和 Stage 17 freshness/degraded |
| 后续阻断 | 未通过时不得宣告 Gate C；不得开始依赖天气提示的 Stage 30，也不得把天气作为地图/行前须知的可信输入 |
| 默认流量影响 | Stage 16 默认 new 后，普通规划请求可使用天气决策；不改变 API schema，只改变候选筛选、PrimaryDraft 与后续重建结果 |

## 2. 当前代码基线与差异

- Stage 4 已规划 `AmapService.get_weather(request_id, city, requested_dates)`，可按请求日期返回 `WeatherDay`，预报范围外为 `unavailable`；Stage 18 前这些天气不得影响候选、Prompt、日期或排序。
- Stage 7A 当前候选池没有室内/室外标签；Stage 7B 明确不向筛选 Prompt 提供天气；Stage 8 仅透传 `weather_context` 且测试锁定天气零影响。
- Stage 13D 已定义点位变化后完整 N 日与递减前缀重建规则；本 Stage 必须复用该重建边界，不允许只替换前端展示或拼接旧路线/时间轴。
- Stage 17 可能让天气来源为 `live/cached_fresh/cached_stale`；`cached_stale` 可触发 `degraded=true`，但不放宽天气真实性或 accepted constraints。

## 3. 文件变更清单

| 操作 | 文件 | 本 Stage 职责 | 完成判据 |
|---|---|---|---|
| 修改 | `backend/app/models/research.py`、`models/source.py` | 为主要点位候选增加 `indoor_outdoor`、`weather_sensitivity`、`weather_tag_quality`、`weather_reasons` | sourced/estimated/unknown 语义稳定，不伪装事实 |
| 新增 | `backend/app/services/weather_context.py` | 将 `WeatherDay` 转为规划可用的脱敏天气上下文、恶劣天气标签和可用范围 | 超范围/缺失为 unavailable，不生成气候判断 |
| 修改 | `backend/app/services/candidate_pool.py` | 在 7A 候选池中补充室内/室外标签：高德字段为 sourced，规则/Agent 推断为 estimated，无法判断为 unknown | unknown 不被当作室内或室外 |
| 修改 | `backend/app/agents/destination_research_agent.py` | 扩展 7B 主要点位筛选输入，允许基于天气从已有候选 ID 中筛选/替换 | 输出仍只含候选 ID 与理由，不含新事实字段 |
| 修改 | `backend/app/agents/itinerary_planner_agent.py` | 扩展 8 PrimaryDraft 输入，允许引用天气来源调整日期/顺序和理由 | 不输出天气事实、酒店、便餐、路线、预算或最终计划 |
| 修改 | `backend/app/coordinators/trip_coordinator.py` | 天气影响点位集合/日期/顺序时，从 PrimaryDraft 起重建 9A～13D 全部下游 | 旧住宿/餐饮/路线/时间轴/预算不得复用 |
| 修改 | `backend/app/services/trip_assembler.py` | 将被正式使用的天气来源纳入 degraded 汇总和脱敏诊断 | stale 天气可 degraded，unavailable 不等于 stale |
| 新增 | `backend/tests/unit/services/test_weather_context.py` | 覆盖预报范围、恶劣天气分类、unknown 和脱敏 | 离线、固定时钟 |
| 新增 | `backend/tests/integration/test_weather_driven_replan.py` | 覆盖天气筛选、PrimaryDraft 重排、下游重建和错误分流 | FakeWeather/FakeLLM/FakeAmapProvider |
| 新增 | `doc/opt_strategy/evidence/stage_18/` | 记录天气范围、室内外标签、重建、测试和回滚证据 | 不提交完整 Prompt、自由文本或原始天气 payload |

不新增前端组件；Stage 14/16 既有状态和 degraded 展示继续消费正式 TripPlan。

## 4. 输入、输出与天气契约

### 4.1 天气上下文

| 输入 | 规则 |
|---|---|
| `WeatherDay` | 只接受 Stage 4/17 的结构化结果；必须有日期、来源、freshness 和字段级 quality |
| `requested_dates` | 只覆盖请求行程 1～3 日；超出 Provider 预报范围的日期为 `unavailable` |
| `severe_weather_flags` | 由稳定规则从 sourced 天气文本/枚举推导，如 heavy_rain、storm、snow、extreme_heat；无法识别时为空 |
| `forecast_scope` | 记录可参与决策的日期集合、unavailable 日期和原因 |

- 天气字段缺失、超范围或 Provider 合法无预报时，只能标记 `unavailable`，不得使用季节、城市气候、常识或 LLM 猜测填充。
- `cached_stale` 天气可参与决策，但必须让最终 TripPlan `degraded=true`；`cached_fresh` 不触发 degraded。

### 4.2 室内/室外标签

| 标签 | 含义 | 来源质量 |
|---|---|---|
| `indoor` | 候选主要活动主要在室内完成 | 高德明确字段为 `sourced`；规则/Agent 判断为 `estimated` |
| `outdoor` | 候选主要活动主要在室外完成 | 高德明确字段为 `sourced`；规则/Agent 判断为 `estimated` |
| `mixed` | 室内外均显著存在，天气只作弱偏好 | `sourced` 或 `estimated` |
| `unknown` | 无法可靠判断 | `unavailable` 或 `estimated_unknown`，不得被规则当作 indoor/outdoor |

- 规则判断只可使用候选已有类型、名称和来源分类，不可调用新外部服务，不可引入自由文本。
- Agent 判断只输出候选 ID 到标签的受控枚举和简短理由；不得改写名称、地址、坐标、评分、开放时间或天气事实。

### 4.3 两次草案顺序

1. TripCoordinator 获取天气上下文，并在 Stage 7A 候选池形成后补充室内/室外标签。
2. Stage 7B 的 DestinationResearchAgent 接收候选 ID、标签、accepted constraints、天气摘要和预报日期，只能筛选/替换已有候选 ID。
3. Stage 8 的 ItineraryPlannerAgent 基于更新后的主要点位集合进行分天与排序，可在理由中引用天气来源和日期。
4. 若主要点位集合、日期或顺序与无天气路径不同，必须从 PrimaryDraft 起重新执行 9A、9B、10A/10B、11、12A、12B、13A～13D。
5. 若天气全 unavailable 或不构成影响，结果可与无天气路径一致，但必须有测试证明天气未被编造。

### 4.4 错误与业务分流

- 天气 Provider 技术错误且无可用 fresh/stale 输入时，按 Stage 15 返回技术错误，不生成带伪天气的计划。
- 天气合法 unavailable 不阻断规划；它只意味着天气不参与决策。
- 天气导致室外候选被替换后，accepted constraints、required POI、美食下限和修复上限仍不可放宽。
- Agent 返回未知候选 ID、unknown 被硬判为 indoor/outdoor、天气理由引用不存在日期或事实时，按核心输出非法重试，耗尽后走 Stage 15 `core_output_invalid`。

## 5. 配置与功能开关

不新增 Settings、环境变量或功能开关。沿用 Stage 4 Provider 配置、Stage 17 天气 TTL 和 Stage 16 `TRIP_PLANNER_MODE`。

受测试锁定的代码常量包括：恶劣天气稳定枚举、室内/室外规则表版本、Agent 重试次数和天气摘要字段白名单；不得通过环境变量制造可漂移业务语义。

## 6. 实施步骤

1. 固化天气上下文模型、恶劣天气枚举、forecast scope 和脱敏日志字段。
2. 扩展候选池模型与室内/室外标签服务，先覆盖 sourced、estimated、unknown 和 mixed。
3. 扩展 DestinationResearchAgent 主要点位筛选输入/输出 schema，确保只返回候选 ID 与天气选择理由。
4. 扩展 PrimaryDraft 输入，让 Stage 8 可基于天气调整日期/顺序并记录来源理由。
5. 在 Coordinator 中接入“天气影响后从 PrimaryDraft 起重建全部下游”的失效边界，禁止复用旧住宿、便餐、路线、时间轴或预算。
6. 补齐 FakeWeather/FakeLLM/FakeAmapProvider 测试、日志脱敏、回滚和 Gate C 输入证据。

## 7. 明确不做

- 不新增 WeatherService，不调用气候服务，不使用气候常识冒充预报，不实现天气预警订阅。
- 不改变 API schema、前端布局、地图、PDF、SSE、行前须知或对话修订。
- 不让天气删除 required POI、放宽 accepted constraints、美食下限、路线真实性、时间窗或预算规则。
- 不让 Agent 新增 POI、改写天气事实、输出最终 TripPlan 或直接修改下游住宿/餐饮/路线。

## 8. 自动化测试

| 用例 ID | 层级 | 场景 | 预期 |
|---|---|---|---|
| S18-T01 | unit | 预报范围内/外 | 范围内可参与决策，范围外 unavailable |
| S18-T02 | unit | 恶劣天气分类 | 稳定 flag，无匹配不猜测 |
| S18-T03 | unit | 室内/室外 sourced/estimated/unknown | unknown 不被当作 indoor/outdoor |
| S18-T04 | contract | 7B Agent 返回未知 ID | 拒绝并重试，耗尽为 core_output_invalid |
| S18-T05 | integration | 暴雨替换室外可选点 | 只选已有候选 ID，required 不被删除 |
| S18-T06 | integration | 仅调整日期/顺序 | PrimaryDraft 变化，9A～13D 下游全部重建 |
| S18-T07 | integration | 有预报但无需调整 | 结果可不变，理由不编造影响 |
| S18-T08 | integration | 全部天气 unavailable | 天气不参与决策，不阻断规划 |
| S18-T09 | integration | 天气 Provider timeout 无 stale | Stage 15 error envelope，无 TripPlan |
| S18-T10 | integration | cached_stale 天气成功 | 规划可继续且 `degraded=true` |
| S18-T11 | regression | Stage 7A/7B/8 边界 | 无酒店/便餐/路线/预算越界字段 |
| S18-T12 | regression | 下游重建 | 点位变化后旧路线、时间轴、预算复用次数为 0 |

```text
python -m pytest backend/tests/unit/services/test_weather_context.py backend/tests/integration/test_weather_driven_replan.py -q
python -m pytest backend/tests -q
npm --prefix frontend run build
```

## 9. 人工验收

| 用例 ID | 步骤 | 预期 |
|---|---|---|
| S18-M01 | 使用暴雨 fixture 规划含室外可选点行程 | 替换/调整理由引用真实天气，页面不显示伪天气 |
| S18-M02 | 使用超预报范围日期 | 天气 unavailable，规划不因气候常识变化 |
| S18-M03 | 使用 stale 天气 fixture | 结果 degraded 提示存在，plan_status 不被改写 |

## 10. 量化退出指标

- [ ] S18-T01～T12、S18-M01～M03 通过，三条命令退出码为 0。
- [ ] 天气导致未知候选 ID、required 删除、accepted 放宽、unknown 强判 indoor/outdoor 的次数为 0。
- [ ] 天气变化后旧住宿/便餐/路线/时间轴/预算被复用的次数为 0。
- [ ] 超范围/缺失天气被伪装为真实预报、气候建议或 LLM 天气事实的次数为 0。
- [ ] 日志、证据和 Prompt 中 Key、完整自由文本、完整天气 raw payload 命中数为 0。

## 11. 迁移与回滚

- 迁移顺序：天气上下文与标签模型 → 7A 标签 → 7B 筛选 → 8 PrimaryDraft → Coordinator 下游重建 → degraded 汇总复核。
- 回滚先关闭 Coordinator 天气输入，再撤回 8/7B/7A 的天气字段消费，最后删除 weather_context 模块；保留 Stage 4 天气展示与 Stage 17 缓存能力。
- 回滚后运行 S18-T04～T10、全量后端和前端 build，确认 Stage 18 前天气零影响测试恢复。

## 12. 交付证据

| 证据 | 路径 | 最小内容 |
|---|---|---|
| 自动化摘要 | `doc/opt_strategy/evidence/stage_18/test-summary.md` | 命令、12 用例、离线声明、失败数 |
| 天气范围矩阵 | `doc/opt_strategy/evidence/stage_18/weather-scope-matrix.md` | 日期、forecast scope、unavailable、freshness |
| 室内外标签矩阵 | `doc/opt_strategy/evidence/stage_18/indoor-outdoor-matrix.md` | sourced/estimated/unknown/mixed、测试 ID |
| 重建矩阵 | `doc/opt_strategy/evidence/stage_18/rebuild-matrix.md` | 点位变化、重建阶段、旧下游失效证明 |
| 人工/回滚 | `doc/opt_strategy/evidence/stage_18/manual-check.md`、`rollback-check.md` | M01～M03、逆序、命令、结论 |

## 13. Ready/Done 复核

当前设计状态：**R2～R10 已细化并通过 Stage 18～20 跨 Stage 一致性审计；执行 Ready 待 R1**。R1 必须等待 Stage 4、13D、16、17 实际 Done，并在开工前复核天气来源、缓存 freshness、7A/7B/8 输入输出、13D 重建和 Stage 15/16 错误/状态契约。当前不表示 Ready、Done、Gate C、设计冻结或允许修改工程代码。
