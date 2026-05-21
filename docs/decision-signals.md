# AI 建议池 / 决策信号语义契约

本文档定义 `DecisionSignal` / `AISuggestion` 的 P0 语义边界，用于后续把现有报告、Agent 输出、告警和后验验证中的操作建议沉淀为结构化信号资产。

当前阶段只定义契约，不改变现有报告展示、通知格式、回测流程、告警中心、持仓管理或真实交易行为。后续 P1/P2 落地存储与提取时必须继续遵守本文档。

## 目标与非目标

目标：

- 把报告中的操作建议、价格计划、风险理由、置信度和证据摘要变成可查询、可反馈、可验证的结构化对象。
- 支持按股票、市场、市场阶段、动作、来源、状态、时间窗口和后验表现聚合分析。
- 保留与历史报告、任务、trace、告警触发和数据质量摘要的关联。
- 允许不完整计划入库，但必须用 `plan_quality` 标注完整度，不得编造缺失价格。

非目标：

- `DecisionSignal` 不是订单，不执行自动下单、券商交易或自动调仓。
- `DecisionSignal` 不保证投资收益，不替代用户独立判断。
- 初期不要求所有模型一次性稳定输出新 schema。
- 不删除或替代现有 Markdown 报告、JSON report、通知文本、回测结果和告警历史。
- 不把命中率简化成单一涨跌判断，必须区分动作、阶段和 horizon。

## 字段边界

建议字段如下，后续实现可按存储约束拆表或扩展，但不得改变字段语义。

| 字段 | 语义 |
| --- | --- |
| `id` | 信号唯一 ID |
| `stock_code` / `stock_name` / `market` | 标的代码、名称和市场 |
| `source_type` | 来源类型：`analysis`、`agent`、`alert`、`market_review`、`manual` |
| `source_agent` | 产出信号的 Agent、策略或模型标签，可为空 |
| `source_report_id` | 关联历史报告 ID，可为空 |
| `trace_id` | 关联一次分析、任务或请求链路的 trace，可为空 |
| `market_phase` | 信号生成时的市场阶段 |
| `trigger_source` | 触发入口：`web`、`api`、`bot`、`schedule`、`action`、`alert_worker` |
| `action` | 标准动作枚举 |
| `action_label` | 面向用户展示的原始或本地化动作文案 |
| `confidence` / `score` | 置信度或评分，需保留来源口径 |
| `horizon` | 建议观察或验证窗口 |
| `entry_low` / `entry_high` | 入场区间，可部分缺失 |
| `stop_loss` | 止损价或风控阈值，可缺失 |
| `target_price` | 目标价，可缺失 |
| `invalidation` | 建议失效条件 |
| `watch_conditions` | 继续观察条件 |
| `reason` | 主要理由 |
| `risk_summary` | 主要风险摘要 |
| `catalyst_summary` | 催化因素摘要 |
| `evidence` | 关联数据、新闻、技术、资金流、持仓成本等证据摘要 |
| `data_quality_summary` | 数据质量、缺失项、降级路径或来源限制 |
| `plan_quality` | 价格计划完整度 |
| `status` | 生命周期状态 |
| `expires_at` | 建议过期时间，可由 `horizon` 或阶段规则推导 |
| `created_at` / `updated_at` | 创建和更新时间 |
| `metadata` | 扩展信息，必须脱敏 |

## 动作 taxonomy

`action` 只能使用以下枚举：

| action | 语义 | 典型用户状态 |
| --- | --- | --- |
| `buy` | 建立新仓或首次买入计划 | 空仓或计划建仓 |
| `add` | 在已有仓位上加仓 | 已持仓 |
| `hold` | 继续持有，不新增仓位 | 已持仓 |
| `reduce` | 部分减仓或降低敞口 | 已持仓 |
| `sell` | 清仓或退出 | 已持仓 |
| `watch` | 观察等待，不直接行动 | 空仓或持仓均可 |
| `avoid` | 暂不参与或规避 | 空仓为主，也可提示持仓风险 |
| `alert` | 风险或条件提醒 | 空仓或持仓均可 |

动作推断规则：

- 空仓语义优先使用 `buy`、`watch`、`avoid`、`alert`。
- 持仓语义优先使用 `add`、`hold`、`reduce`、`sell`、`alert`。
- 若报告只给出“关注”“观察”“等待突破”等非交易建议，使用 `watch`，不得强行映射为 `buy`。
- 若报告只给出“风险升高”“触发风控”“需警惕”等提醒，使用 `alert` 或 `reduce` / `sell`，具体取决于是否明确要求减仓或退出。
- `action_label` 可以保留报告原文，例如“震荡观望”“洗盘观察”“逢低布局”，但 `action` 必须归一到标准枚举。

## 市场阶段

`market_phase` 用于避免盘前计划、盘中提醒和盘后复盘混用语义：

| market_phase | 语义 |
| --- | --- |
| `premarket` | 盘前计划或开盘前准备 |
| `intraday` | 盘中观察、条件触发或实时提醒 |
| `postmarket` | 盘后复盘、次日计划或中短期建议 |
| `non_trading` | 非交易日、休市或无法归入盘中交易阶段 |
| `unknown` | 无法可靠判断阶段 |

阶段影响：

- `premarket` 侧重计划和条件，不应默认为盘中已触发。
- `intraday` 侧重短期条件和风险提醒，默认有效期更短。
- `postmarket` 侧重复盘和未来窗口验证，可按 `1d`、`3d`、`5d`、`10d` 等 horizon 评估。
- `unknown` 只能作为保守 fallback，后续实现应记录数据来源和判断失败原因。

## Horizon

`horizon` 表示建议适用或后验验证窗口：

| horizon | 语义 |
| --- | --- |
| `intraday` | 当日盘中 |
| `1d` | 下一个交易日或一个交易日窗口 |
| `3d` | 三个交易日窗口 |
| `5d` | 五个交易日窗口 |
| `10d` | 十个交易日窗口 |
| `swing` | 波段周期，需结合实现定义最大观察窗口 |
| `long` | 长线周期，需结合实现定义复核频率 |

若报告没有明确 horizon，提取器应根据来源和阶段保守推断，并在 `metadata` 或 `data_quality_summary` 中记录推断来源。不得为了提高完整度编造具体日期或价格。

## Plan Quality

`plan_quality` 用于标注价格计划完整度，而不是判断建议好坏：

| plan_quality | 条件 |
| --- | --- |
| `complete` | 同时包含入场区间、止损、目标价、失效条件和观察条件 |
| `actionable` | 至少包含动作、理由、风险摘要，并有入场区间或止损/目标价中的关键风控信息 |
| `partial` | 有动作和理由，但价格计划或风控条件明显不完整 |
| `watch_only` | 仅适合作为观察或提醒，不构成交易计划 |
| `insufficient` | 信息不足，只能保留为低质量候选或跳过写入 |

缺字段处理规则：

- 缺少 `entry_low` / `entry_high` 时不填入臆测价，只降低 `plan_quality`。
- 缺少 `stop_loss` 或 `target_price` 时不从支撑/压力位强推，除非报告已经明确给出可解释的映射。
- 价格字段必须是可解析的数值或空值，不保存“附近”“一带”等模糊文本为价格字段；原文可进入 `reason`、`watch_conditions` 或 `metadata.raw_text_excerpt`。
- 明显不合法价格计划，例如负数价格、入场下界高于上界、止损高于目标价且无做空语义，应标记为低质量或跳过，并记录原因。

## 生命周期状态

`status` 只能使用以下枚举：

| status | 语义 |
| --- | --- |
| `active` | 当前有效，可查询和复用 |
| `expired` | 已超过有效期，未必代表建议错误 |
| `invalidated` | 触发失效条件或出现相反信号 |
| `closed` | 已完成验证、用户关闭或策略结束 |
| `archived` | 归档保留，不参与默认最新查询 |

生命周期规则：

- 新信号默认 `active`，除非提取时已判断过期或信息不足。
- `expires_at` 可以来自报告明示时间，也可以按 `horizon` 和 `market_phase` 推导。
- 盘中 `intraday` 信号默认到当日收盘或本轮交易日结束。
- 盘前 `premarket` 计划通常到当日收盘或下一个交易日。
- 盘后 `postmarket` 建议通常按 `1d`、`3d`、`5d`、`10d` 等窗口过期。
- 新的相反信号出现时，后续实现可将旧信号标记为 `invalidated`，或保留两者并记录冲突原因。

## 去重与关联

后续写入层必须避免同一报告、同一股票、同一动作重复生成多条近似信号。建议去重键至少考虑：

- `source_report_id`
- `stock_code`
- `action`
- `market_phase`
- `horizon`
- 价格计划摘要或归一化后的理由摘要

信号应尽量关联历史报告、trace、触发来源、市场阶段和股票。缺少关联信息时可以写入不完整信号，但需要在 `data_quality_summary` 中标注限制。

## 脱敏与安全边界

信号池不得保存以下敏感信息：

- token、cookie、session、JWT、API key
- webhook URL、机器人 token、邮箱密码、邮箱授权码
- 数据源、券商或第三方平台凭据
- 用户本地绝对路径、私有内网地址，除非已经做脱敏或 hash

`metadata`、`evidence` 和 `data_quality_summary` 必须经过脱敏后再持久化。若无法确认内容安全，应只保存字段级摘要，不保存原始 payload。

## 后验验证口径

后续验证信号表现时，至少需要区分：

- 动作：`buy/add/hold/reduce/sell/watch/avoid/alert`
- 市场：A 股、港股、美股等
- 阶段：`premarket`、`intraday`、`postmarket` 等
- horizon：`intraday`、`1d`、`3d`、`5d`、`10d` 等
- 来源：报告、Agent、告警、人工等
- 数据质量：完整、降级、缺价、停牌、非交易日、无法评估

无法评估时应输出原因，例如缺少后续价格、停牌、非交易日、标的退市、数据源失败或价格计划不完整。不得把无法评估直接计为失败或成功。

## 兼容、配置与回滚

- 当前 P0 不新增环境变量，不需要更新 `.env.example` 或 config registry。
- 当前 P0 不新增 API、Repository、数据库表或 Web 页面。
- 当前 P0 不改变默认报告展示、通知格式、历史记录、回测和告警行为。
- 若后续实现提供写入开关，关闭信号提取或信号写入后，现有分析流程必须继续正常运行。
- 回滚本 P0 文档只会移除语义契约入口，不影响运行时代码。
