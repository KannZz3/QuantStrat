# Market Anomalies -- 市场异象

本目录研究中国商品期货市场中的技术分析异象：先复刻“高波动资产中技术分析更有效”的横截面思想，再检验该异象是否会在高流动性/高交易活跃度状态下衰减，最后考察样本内筛选出的策略在样本外和 pseudo-live 阶段是否失效。

理论框架主要来自三篇文献：

- Han, Yang, and Zhou (2013): 技术分析策略在高波动资产组合中可能具有更强横截面盈利能力。
- Chordia, Subrahmanyam, and Tong (2014): 高流动性、高交易活跃度和更强套利活动会削弱资本市场异象。
- McLean and Pontiff (2016): 被研究、发表和交易后的收益可预测性会在样本外及发表后衰减。

## Data

基础数据为 `01_base_clean_panel.zip` 中的 `01_base_clean_panel.csv`：

- 样本区间：2016-01-05 至 2026-06-18
- 样本规模：58,591 行
- 标的数量：27 个商品期货品种
- 行业数量：9 个 sector
- 主要字段：OHLC、volume、open interest、turnover value、日收益、forward return、可交易标记、异常收益标记

所有回测均使用延迟执行原则：信号在 `t-1` 形成，仓位在 `t` 执行，收益窗口为 `close[t] -> close[t+1]`。

## Workflow

### 1. Volatility-Sorted Technical Anomaly

代码：`02_vol_sorted_technical_anomaly.ipynb`

目的：检验 Han, Yang, and Zhou 式的“波动率排序技术分析异象”是否存在于商品期货中。

流程：

- 计算每个品种的 `RV60`。
- 每日按横截面 RV60 分成 `Low / Mid / High` 波动率桶。
- 构造 MA10、MA20、MA60、MA120 技术信号。
- 分别测试 `long_flat` 和 `long_short`。
- 计算每个波动率桶的等权策略收益。
- 构造 `High - Low` 波动率桶 HML。
- 输出单调性与 HML 统计。

主要结果文件：

- `02_bucket_performance_summary.csv`
- `02_high_minus_low_summary.csv`
- `02_monotonic_flags.csv`

### 2. Liquidity / Activity Attenuation

代码：`03_liquidity_attenuation.ipynb`

目的：检验技术分析异象是否在高交易活跃度状态下变弱。

流程：

- 使用 volume、open interest、turnover value 构造 activity proxy。
- 对三个活跃度指标做 symbol-level rolling z-score。
- 合成 `activity_score`，并分为 `Low / Mid / High` activity bucket。
- 将 signal-date activity 合并到 Notebook 02 的策略收益中。
- 在每个 activity bucket 内重新计算 High-vol minus Low-vol HML。
- 计算 `High Activity HML - Low Activity HML`。
- 估计 interaction regression: return ~ high_vol + activity_score + high_vol x activity_score。

主分析只使用 `signal_date` 的 activity 状态，不使用 execution-date activity，以避免时间泄露。

主要结果文件：

- `03_hml_by_activity_summary.csv`
- `03_activity_attenuation_summary.csv`
- `03_activity_interaction_regression.csv`
- `03_interaction_flags.csv`
- `03_activity_flags.csv`

### 3. Post-Selection Decay

代码：`04_post_selection_decay.ipynb`

目的：检验样本内筛选出的技术策略，在样本外和 pseudo-live 阶段是否衰减。

流程：

- 构造 26 个候选策略，覆盖 MA、TSMOM、BREAKOUT、REVERSAL。
- 每个策略均测试 `long_flat` 和 `long_short`。
- 样本切分：
  - IS: 2016-01-05 至 2020-12-31
  - OOS: 2021-01-01 至 2023-12-31
  - PSEUDO_LIVE: 2024-01-01 以后
- 样本切分使用完整 return window，避免跨区间收益污染。
- 仅用 IS 期间的 `net_strategy_return` 计算 selection score。
- 选择 Top 10 策略，比较其 IS / OOS / PSEUDO_LIVE 表现。

主要结果文件：

- `04_strategy_performance_by_split.csv`
- `04_in_sample_selection_scores.csv`
- `04_selected_strategies.csv`
- `04_all_strategy_decay_summary.csv`
- `04_post_selection_decay_summary.csv`

## Main Findings from CSV Results

### Finding 1: Volatility-sorted technical anomaly is partial, not strong

基于 `02_high_minus_low_summary.csv` 和 `02_monotonic_flags.csv`：

- `net_strategy_return` 的 High-minus-Low HML 为正：5 / 8。
- 显著正 HML (`t > 1.96`)：0 / 8。
- 单调性结果：
  - `PARTIAL`: 5
  - `REVERSED`: 3
  - `STRONG`: 0

最强的正向结果集中在 MA20：

- MA20 long_short: HML 年化收益 5.27%，t = 0.91。
- MA20 long_flat: HML 年化收益 4.30%，t = 0.96。

长窗口 MA120 反而出现反转：

- MA120 long_flat: HML 年化收益 -3.44%，t = -0.77。
- MA120 long_short: HML 年化收益 -10.23%，t = -1.77。

结论：商品期货中存在局部 Han-Yang-Zhou 式高波动技术分析优势，尤其在 MA20 附近；但该异象不强、不显著，也不稳定。

### Finding 2: Activity attenuation is directionally present but statistically weak

基于 `03_activity_flags.csv`：

- `WEAK_ATTENUATION`: 18 / 24。
- `WEAK_ENHANCEMENT`: 6 / 24。
- bucket attenuation 的 `|t| > 1.96`: 0 / 24。
- interaction regression 的 `|t| > 1.96`: 0 / 24。

只看 `net_strategy_return`：

- MA10: 两个版本为 weak enhancement。
- MA20、MA60、MA120: 多数为 weak attenuation。

长窗口的衰减方向最明显。例如 MA120 long_short：

- Low-activity HML 年化收益：1.20%。
- High-activity HML 年化收益：-16.44%。
- High-minus-Low activity attenuation：-22.24%，t = -1.47。

结论：高交易活跃度状态下，技术异象多数方向上变弱，符合 Chordia, Subrahmanyam, and Tong 的机制直觉；但统计显著性不足，因此只能表述为弱证据或方向性证据。

### Finding 3: Post-selection decay is the strongest result

基于 `04_selected_strategies.csv` 和 `04_post_selection_decay_summary.csv`：

Top 10 IS-selected strategies 的平均表现：

| Split | Mean annual return | Mean Sharpe |
| --- | ---: | ---: |
| IS | 5.96% | 0.68 |
| OOS | 3.72% | 0.35 |
| PSEUDO_LIVE | 1.06% | 0.11 |

Top 10 衰减分类：

- `SEVERE_DECAY`: 5
- `PARTIAL_DECAY`: 3
- `DECAYED_TO_NEGATIVE`: 2
- `ROBUST`: 0

中位保留率：

- OOS return retention: 66.6%。
- PSEUDO_LIVE return retention: 14.4%。
- OOS Sharpe retention: 55.5%。
- PSEUDO_LIVE Sharpe retention: 16.6%。

代表性策略：

- `BREAKOUT_20_long_flat`: IS 年化 8.93%，OOS 4.93%，PSEUDO_LIVE 3.55%，分类为 `PARTIAL_DECAY`。
- `MA_20_long_flat`: IS 年化 7.06%，OOS 4.96%，PSEUDO_LIVE 0.01%，分类为 `SEVERE_DECAY`。
- `MA_10_long_flat`: IS 年化 5.74%，OOS 2.89%，PSEUDO_LIVE -0.19%，分类为 `DECAYED_TO_NEGATIVE`。

结论：样本内筛选出的技术策略在 OOS 仍保留部分收益，但到 pseudo-live 阶段大幅衰减。这是本项目中最强、最一致的结果，和 McLean and Pontiff 的 post-discovery / post-publication decay 思想最接近。

## Interpretation

本项目的结果应谨慎表述为：

1. 商品期货中存在局部波动率排序技术分析异象，但没有强显著证据。
2. 高交易活跃度状态下，异象多数方向上变弱，但衰减证据主要是弱统计证据。
3. 样本内筛选出的技术策略在样本外，尤其 pseudo-live 阶段显著衰减，这是最稳健的发现。

因此，当前证据更支持以下判断：

> 商品期货技术分析异象更像是阶段性、可被筛选和交易拥挤侵蚀的收益来源，而不是稳定、强显著、长期不变的风险溢价。

## Limitations

- Notebook 02 和 Notebook 03 的主要结论不应夸大为强显著发现。
- Notebook 04 是 post-selection decay，不是严格的 post-publication event study。
- Activity proxy 假设 signal-date 的 volume、open interest 和 turnover value 在下一交易日执行前可得。
- 所有结果均基于当前 CSV 输出，不构成投资建议。
