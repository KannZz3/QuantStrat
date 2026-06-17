# 郑商所白糖主力合约 EGARCH(1,1)-t 波动率建模

本项目基于郑商所白糖主力合约日收益率，构建一套完整的波动率建模与风险管理框架，覆盖收益率检验、GARCH 类模型筛选、残差诊断、波动率预测、VaR 回测、仓位缩放，以及 Hong-Lee generalized spectral volatility adequacy test 模型健康检验。

本项目不是方向预测模型，不判断白糖应上涨还是下跌。项目目标是建立一个可复用的风险状态判断模块，用于回答：

* 当前白糖处于低波动、正常波动还是高波动状态；
* 下一交易日正常风险范围有多宽；
* 多头 / 空头单日 VaR 风险边界在哪里；
* 当前仓位是否应因波动率变化而放大或缩小；
* EGARCH(1,1)-t 是否充分刻画白糖收益率的厚尾和波动率聚集；
* Hong-Lee 检验是否提示模型存在遗漏的广义波动率依赖。

---

## 1. Project Scope

### Covered

* 白糖主力合约日收益率建模；
* ARCH / GARCH / GJR-GARCH / EGARCH 候选模型比较；
* Student-t 厚尾误差分布；
* 标准化残差诊断；
* EGARCH 条件波动率预测；
* 单日多空 VaR 风险边界；
* Kupiec VaR 无条件覆盖率回测；
* 目标波动率仓位缩放；
* Hong-Lee generalized spectral volatility adequacy test；
* project-level data-driven bandwidth robustness；
* Monte Carlo size / power debug module。

### Not Covered

* 不预测价格方向；
* 不生成做多 / 做空信号；
* 不直接给出止损位；
* 不替代基本面、价差结构、库存、进口、天气、外盘原糖或成交持仓分析；
* 当前 Monte Carlo 仅为 debug output，不构成正式 size / power 统计结论。

---

## 2. Data and Preliminary Diagnostics

样本为郑商所白糖主力合约日频数据，共 2536 个交易日。收益率描述统计如下：

| Item              |    Value |
| ----------------- | -------: |
| Sample size       |     2536 |
| Mean daily return | -0.0018% |
| Daily return std  |  0.8407% |
| Skewness          |   0.0552 |
| Excess kurtosis   |   3.8713 |

收益率均值接近 0，偏度接近 0，说明收益率分布大致对称；超额峰度为 3.8713，对应普通峰度约 6.8713，明显高于正态分布的普通峰度 3，说明收益率存在明显厚尾。

ADF 检验显示，对数价格序列不平稳，收益率序列显著平稳：

| Series    | ADF p-value | Decision                               |
| --------- | ----------: | -------------------------------------- |
| log price |      0.2401 | non-stationary / insufficient evidence |
| return    |    ≈ 0.0000 | stationary                             |

因此，本项目对日收益率 `ret_pct` 建模，而不是直接对价格或对数价格建模。

传统依赖检验显示，收益率本身线性自相关不强，但收益率平方在 lag 20、lag 30 附近存在显著自相关，ARCH-LM 在 lag 20 下显著。这说明白糖收益率方向自相关不稳定，但波动率存在阶段性聚集，适合使用 GARCH 类模型刻画条件波动率。

---

## 3. Model Selection and Final EGARCH Specification

项目比较以下候选模型：

* ARCH(5)-t
* GARCH(1,1)-Normal
* GARCH(1,1)-t
* AR(1)-GARCH(1,1)-t
* GJR-GARCH(1,1)-t
* EGARCH(1,1)-t

在本文设定的诊断滞后阶数和 5% 显著性水平下，所有候选模型均未显示明显剩余自相关或剩余 ARCH 效应。具体表现为：标准化残差无明显线性自相关，标准化残差平方无明显剩余自相关，ARCH-LM 未发现明显剩余 ARCH 效应。

在通过残差诊断的模型中，EGARCH(1,1)-t 的 AIC 和 BIC 最低：

| Model         |     AIC |     BIC |
| ------------- | ------: | ------: |
| EGARCH(1,1)-t | 5969.51 | 6004.54 |

因此，本项目选择以下模型作为主模型：

```text
Constant Mean + EGARCH(1,1) + Student-t error
```

关键参数如下：

| Parameter  | Estimate | Interpretation               |
| ---------- | -------: | ---------------------------- |
| `mu`       | 0.000002 | 均值项接近 0，且不显著；模型不提供方向预测       |
| `alpha[1]` | 0.057641 | 冲击大小项显著，大涨大跌会影响后续波动率         |
| `gamma[1]` | 0.019412 | 非对称项在 5% 水平下不显著，仅能说明存在弱非对称迹象 |
| `beta[1]`  | 0.989908 | 波动率持续性极强，高波动和低波动状态都会延续       |
| `nu`       | 4.497111 | Student-t 自由度较低，收益率厚尾明显      |

模型结论：

* 白糖收益率存在明显厚尾；
* 条件波动率具有很强持续性；
* 冲击大小会显著影响未来波动率；
* 非对称波动迹象存在，但 5% 水平下证据不强；
* Student-t 误差分布比 Normal 分布更适合白糖收益率。

---

## 4. Residual Diagnostics and Current Volatility State

EGARCH(1,1)-t 标准化残差诊断结果如下：

| Diagnostic                            |   Value |
| ------------------------------------- | ------: |
| Standardized residual mean            | -0.0067 |
| Standardized residual std             |  0.9905 |
| Standardized residual skew            | -0.1431 |
| Standardized residual excess kurtosis |  3.8190 |
| ARCH-LM lag 10 p-value                |  0.8691 |
| Ljung-Box z lag 10 p-value            |  0.3220 |
| Ljung-Box z² lag 10 p-value           |  0.8621 |
| Ljung-Box z² lag 20 p-value           |  0.8152 |
| Ljung-Box z² lag 30 p-value           |  0.9543 |
| Traditional decision                  |    PASS |

EGARCH(1,1)-t 基本解释了原始收益率中的条件异方差结构。标准化残差本身无明显线性自相关，标准化残差平方无明显剩余波动率聚集，ARCH-LM 未发现剩余 ARCH 效应。标准化残差超额峰度仍大于 0，对应普通峰度约 6.8190，说明极端冲击仍然存在，这进一步支持 Student-t 厚尾分布设定。

截至最新样本日，当前波动率状态如下：

| Item                             |               Value |
| -------------------------------- | ------------------: |
| Latest date                      | 2026-06-16 16:00:00 |
| Latest close                     |                5343 |
| Latest return                    |             0.4878% |
| EGARCH volatility                |             0.6445% |
| Historical volatility percentile |               8.24% |
| Volatility state                 |             low_vol |
| Next-day forecast volatility     |             0.6516% |

当前白糖处于低波动压缩区。条件波动率仅位于历史 8.24% 分位，说明当前不是高波动状态，也不是异常冲击状态，而是历史偏低波动环境。

未来 10 个交易日预测波动率从 0.6516% 小幅回升至 0.6699%，对应历史分位从约 8.79% 升至约 10.80%。短期波动率有温和修复迹象，但仍处于低波动区间。

---

## 5. VaR, Backtesting, and Position Scaling

基于 EGARCH(1,1)-t 和 Student-t 分布，下一交易日 VaR 结果如下：

| Risk measure               | Return threshold |
| -------------------------- | ---------------: |
| Long 95% VaR lower return  |         -1.0031% |
| Long 99% VaR lower return  |         -1.7131% |
| Short 95% VaR upper return |          1.0031% |
| Short 99% VaR upper return |          1.7131% |

按最新收盘价 5343 换算：

| Confidence             | Price range |
| ---------------------- | ----------: |
| 95% one-day risk range | 约 5289–5397 |
| 99% one-day risk range | 约 5252–5435 |

VaR 用于衡量统计意义下的单日风险范围，不代表价格一定触及，也不能直接等同于止损位。

Kupiec 无条件覆盖率回测显示，多头 95%、多头 99%、空头 95%、空头 99% VaR 均通过 5% 显著性水平检验：

| Side  | Confidence | Actual exception rate | Expected rate | Decision |
| ----- | ---------: | --------------------: | ------------: | -------- |
| Long  |        95% |                 5.36% |         5.00% | Pass     |
| Long  |        99% |                0.946% |         1.00% | Pass     |
| Short |        95% |                 4.73% |         5.00% | Pass     |
| Short |        99% |                0.986% |         1.00% | Pass     |

结论：EGARCH(1,1)-t 对白糖单日尾部风险的无条件覆盖率整体合理，尤其 99% VaR 对极端尾部风险刻画较好。限制是，Kupiec 检验只检查 VaR 例外次数是否合理，不检查例外是否成团，后续可加入 Christoffersen independence / conditional coverage 回测。

项目以历史中位数波动率 0.8323% 作为目标波动率。当前 EGARCH 条件波动率为 0.6445%，低于目标波动率，因此目标波动率法给出的仓位缩放因子为：

| Item                             |   Value |
| -------------------------------- | ------: |
| Target volatility                | 0.8323% |
| Current EGARCH volatility        | 0.6445% |
| Raw position scale               |  1.2914 |
| Hong-Lee final model health      | healthy |
| Hong-Lee final health multiplier |  1.0000 |
| Final position scale             |  1.2914 |

如果已有独立方向信号，且满足保证金、止损、流动性和组合风险约束，目标波动率法给出的风险预算上限约为 1.29 倍标准仓位。但该因子只反映波动率风险预算，不提供方向信号。

模块分工如下：

* 方向信号决定做多还是做空；
* EGARCH 波动率决定仓位放大还是缩小；
* Hong-Lee model health 决定是否对 EGARCH 风险输出打折；
* VaR 边界用于衡量单日尾部风险；
* 标准化残差用于监控异常冲击。

---

## 6. Hong-Lee Volatility Adequacy Test

在传统残差诊断之后，项目加入 Hong and Lee (2017) generalized spectral volatility adequacy test，用于检查 EGARCH(1,1)-t 是否仍遗漏传统 Ljung-Box / ARCH-LM 难以识别的线性、非线性、长滞后或未知形式的剩余广义波动率依赖。

该模块不提供多空方向信号，只用于判断 EGARCH 风险输出是否需要降权。

### Fixed-bandwidth core test

主检验采用单变量 fixed-bandwidth core Hong-Lee M(p) 统计量。

| Item                           |                  Value |
| ------------------------------ | ---------------------: |
| Target model                   |          EGARCH(1,1)-t |
| Main bandwidth p               |                21.0027 |
| Kernel                         |               Bartlett |
| W0 grid                        | 8-point symmetric grid |
| Robust M(p)                    |                -0.3217 |
| Robust p-value                 |                 0.6261 |
| Robust decision                |         FAIL_TO_REJECT |
| IID p-value                    |                 0.7714 |
| Bandwidth stability            |  stable_fail_to_reject |
| Reject share across bandwidths |                   0.00 |
| Model health                   |                healthy |
| Health multiplier              |                 1.0000 |

固定主带宽 p = 21.0027 下，robust p-value = 0.6261，未拒绝 EGARCH(1,1)-t 的波动率模型充分性原假设。多个 bandwidth 稳健性检验也均未拒绝，说明当前结论不是单一 p 选择导致的偶然结果。由于白糖标准化残差仍存在厚尾，项目以 robust M(p) 作为主判断，iid 版本仅作为对照。

### Project-level data-driven bandwidth

项目进一步加入 project-level penalized bandwidth selector，用于降低人为选择单一 bandwidth 的影响。该选择器在多个候选 p 上计算 robust M(p)，并通过惩罚项降低过度选择复杂 bandwidth 的风险。

该模块是面向白糖项目的 project-level penalized selector，不是 Hong and Lee (2017) Section 6 原文 plug-in bandwidth 公式的逐字复刻。

| Item                       |                 Value |
| -------------------------- | --------------------: |
| Fixed main p               |               21.0027 |
| Fixed main decision        |        FAIL_TO_REJECT |
| Sample-selected p          |               15.5400 |
| Sample-selected-p M        |               -0.2863 |
| Sample-selected-p p-value  |                0.6127 |
| Sample-selected-p decision |        FAIL_TO_REJECT |
| Multi-p stability          | stable_fail_to_reject |
| Multi-p reject share       |                  0.00 |
| Final bandwidth conclusion |      bandwidth_robust |

固定主带宽、多候选带宽和 sample-selected p 下，Hong-Lee 检验均未拒绝 EGARCH(1,1)-t 的波动率模型充分性原假设。因此，当前 Hong-Lee 结论对 bandwidth 选择具有稳健性。

需要注意，sample-selected p 的 p-value 属于 post-selection descriptive p-value。由于 p 是基于同一真实样本选择出来的，该 p-value 不能简单等同于固定 p 下的标准检验 p-value。

---

## 7. Monte Carlo Debug Module

为检查 project-level bandwidth selector 和 Hong-Lee model health flag 的有限样本行为，项目加入 Monte Carlo debug module。

当前设置：

| Item             |                            Value |
| ---------------- | -------------------------------: |
| MC_FAST_MODE     |                             True |
| MC_REPS          |                               10 |
| MC_T             |                             2536 |
| MC_BURN          |                              500 |
| MC_ALPHA         |                             0.05 |
| MC_RESELECT_DD_P |                             True |
| CI method        |                           Wilson |
| Boundary         | debugging_only_MC_REPS_below_100 |

Size simulation 使用 fitted EGARCH(1,1)-t 作为 H0 DGP；power simulation 使用 GJR / threshold volatility 与 regime-switching volatility 作为 H1 DGP。

每次 Monte Carlo replication 计算：

* sample-selected-p Hong-Lee；
* re-selected project-level data-driven-p Hong-Lee；
* Ljung-Box(z)；
* Ljung-Box(z²)；
* ARCH-LM。

当前 MC_REPS = 10，仅用于代码调试，不用于正式 size / power 统计结论。因此，所有 size / power 结果统一标记为 `debugging_only`，不解释为 `too_conservative`、`too_aggressive`、`low_power`、`moderate_power` 或 `high_power`。

Monte Carlo 拒绝率置信区间使用 Wilson score interval，而不是 Wald interval。这样可以避免在 0/10 拒绝时将 95% CI 错误写成 [0, 0]。

| Case         | Observed rejection rate |   Wilson 95% CI |
| ------------ | ----------------------: | --------------: |
| 0/10 rejects |                   0.00% | [0.00%, 27.75%] |
| 1/10 rejects |                  10.00% | [1.79%, 40.42%] |

当前 debug summary：

| Section     | Key result                                                         | Interpretation                    |
| ----------- | ------------------------------------------------------------------ | --------------------------------- |
| Size        | re-selected project-dd p: 0/10 rejects                             | debugging_only                    |
| Power       | GJR / regime-switching H1: no Hong-Lee rejects observed in 10 reps | debugging_only                    |
| CI method   | Wilson                                                             | avoids degenerate 0/10 Wald CI    |
| Reliability | MC_REPS = 10                                                       | no formal size / power conclusion |

完整 Monte Carlo 结果保存为：

```text
hong_lee_project_dd_and_monte_carlo_results.xlsx
```

---

## 8. Project Outputs and Usage

主要输出包括：

| Output                                             | Purpose                            |
| -------------------------------------------------- | ---------------------------------- |
| `traditional_model_comparison`                     | 选择主波动率模型                           |
| `egarch_parameter_summary`                         | 汇总 EGARCH 参数                       |
| `egarch_residual_diagnostics`                      | 检查传统残差充分性                          |
| `egarch_volatility_forecast`                       | 预测未来条件波动率                          |
| `egarch_var_results`                               | 给出多空单日风险边界                         |
| `kupiec_var_backtest`                              | 检查 VaR 无条件覆盖率                      |
| `position_scaling_table`                           | 根据波动率调整仓位                          |
| `hong_lee_fixed_bandwidth_results`                 | 固定带宽 Hong-Lee 检验                   |
| `hong_lee_project_dd_results`                      | project-level bandwidth robustness |
| `monte_carlo_size_power_debug_results`             | MC size / power debug              |
| `hong_lee_project_dd_and_monte_carlo_results.xlsx` | Hong-Lee 与 MC 汇总结果                 |

本项目适合用作：

* 白糖主力合约日频风险状态监控；
* 波动率分位识别；
* VaR 单日风险边界计算；
* 目标波动率仓位缩放；
* GARCH 类波动率模型研究模板；
* Hong-Lee 模型充分性检验接入模板；
* 后续滚动样本外 VaR / 波动率预测研究基础。

本项目不适合直接用作：

* 价格方向预测器；
* 自动交易信号；
* 单独止损系统；
* 基本面分析替代品；
* 高频交易模型。

---

## 9. Limitations and Main Conclusion

当前项目边界：

* EGARCH(1,1)-t 是波动率模型，不是方向预测模型；
* Hong-Lee generalized spectral test 是模型充分性检验，不是交易信号；
* 当前 Hong-Lee 未拒绝，只表示未发现显著剩余广义波动率依赖，不等于证明模型绝对正确；
* sample-selected p-value 是 post-selection descriptive；
* project-level bandwidth selector 不是 Hong and Lee (2017) Section 6 的完整复刻；
* 当前 Monte Carlo 仅为 MC_REPS = 10 的 debug output；
* 若用于样本外交易回测，Hong-Lee health 不能用全样本结果回填历史，否则会产生 look-ahead bias。

后续扩展方向：

* 将 MC_REPS 提高至 500 或 1000；
* 加入 Christoffersen VaR independence / conditional coverage 回测；
* 加入滚动样本外 EGARCH 估计和 VaR 预测；
* 复核极端标准化残差日期及对应基本面事件；
* 扩展更多 H1 DGP，例如 long-memory volatility、volatility jumps、stronger regime-switching；
* 进一步复刻 Hong and Lee (2017) Section 6 原始 data-driven bandwidth selection 机制。

主结论：

白糖主力合约日收益率存在明显厚尾，条件波动率具有强持续性。EGARCH(1,1)-t 在传统残差诊断、VaR 无条件覆盖率检验和 Hong-Lee generalized spectral volatility adequacy test 下均未发现明显剩余波动率结构。

截至最新样本日，白糖处于低波动压缩区。若已有独立方向信号，且满足保证金、止损、流动性和组合风险约束，目标波动率法给出的风险预算上限约为 1.29 倍标准仓位。

但本项目不提供多空方向。是否交易、交易方向、止损位置和持仓周期仍需结合基本面、价格结构、成交持仓、风险预算和交易纪律。


