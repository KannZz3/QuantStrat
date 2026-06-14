# BTC 统一多维定价与估值模型项目 (QuantStrat)

本项目是一个量化金融研究与实现项目，实现并优化了一个**比特币（BTC）统一多维动态下沿估值模型**。该模型基于三大经典学术框架（Bhambhwani et al. 2019 / Biais et al. 2023 / Liu & Tsyvinski 2021），对 BTC 链上基本面、市场便利性以及投资者注意力进行多维度量化评估。

---

## 1. 项目是做什么用的？

本项目旨在通过**多源公网数据抓取、严格的学术一致性交叉验证、动态风险折价与压力测试**，计算出比特币在不同压力情景下的合理估值下沿。

模型由以下三个层级串联组成：
1. **核心锚（BDK 链上基本面）**：基于算力（Hashrate）与活跃地址数（Active Addresses），锚定 BTC 长期链上价值。
2. **第一折价层（Biais 均衡收益）**：整合交易便利收益（Transaction Benefit）、交易成本（Transaction Cost）、市场进入渠道（Market Access, ETF）和崩盘风险（Crash Risk）。
3. **第二折价层（Liu-Tsyvinski 风险收益）**：整合市场动量（Momentum）与投资者注意力（Attention / Negative Attention）。

---

## 2. 核心框架与处理流程

项目的数据管道与估值流程严格遵循以下闭环：

```mermaid
graph TD
    A[多源数据并行抓取] --> B[数据清洗与特征构建]
    B --> C[严格交叉验证与自校验]
    C --> D[动态权重评分与折扣层计算]
    D --> E[压力情景下沿估值输出]
    
    subgraph 抓取源 (Public APIs)
        A1[价格: Coinbase/Kraken/Yahoo] --> A
        A2[链上: Blockchain/CoinMetrics] --> A
        A3[注意力: Wikipedia Pageviews] --> A
    end

    subgraph 验证机制 (Validator)
        C1[共识价格/算力/活跃地址验证] --> C
        C2[维基百科单源自校验回退] --> C
    end
```

### 核心设计特点
* **严格交叉验证 (Validator)**：排除手动或不可靠的单一数据源。只有通过双源一致性校验（相关系数、水平差距、分位数差异）的数据才允许进入定价。
* **维基百科单源自校验回退 (Wiki Fallback)**：针对高频受限/封禁的 API（如 Google Trends 和 GDELT），在双源校验失效时，自动启用维基百科单源（非恒定常数且满足观测样本数）自校验逻辑，保障模型不退化。
* **防 NaN 传播动态加权评分**：评分算法采用掩码动态重归一化，当部分可选数据源（如 ETF 净流入或转账量）因 API 缺失时，能自动重新分配权重进行加权平均，防止 NaN 值的级联污染。

---

## 3. 项目结构与模块划分

项目采用高度模块化的结构，便于扩展与维护：

```text
QuantStrat/
├── 加密货币定价&基本面/
│   ├── btc_unified_pricing_model_v1_3.py  # 命令行运行入口
│   ├── requirements.txt                   # 项目依赖
│   ├── config.example.json                # 参数配置文件示例
│   │
│   ├── btc_unified_pricing_model/         # 核心模型包
│   │   ├── __init__.py
│   │   ├── config.py                      # 统一参数配置 (ModelConfig)
│   │   ├── utils.py                       # 抓取重试、Winsorize、Z-score 等通用工具
│   │   ├── fetchers.py                    # 异步/并行多源 API 数据抓取器
│   │   ├── processor.py                   # 字段平滑与时序特征工程
│   │   ├── validator.py                   # 交叉验证核心逻辑 (自校验与回退机制)
│   │   ├── pricing.py                     # BDK 估值锚、折价评分计算与压力情景模拟
│   │   ├── io_outputs.py                  # 多格式报表导出与控制台报告打印
│   │   └── pipeline.py                    # 调度数据流与估值管道的控制器
│   │
│   ├── tests/                             # 单元测试模块
│   │   └── test_core.py                   # 核心计算与验证逻辑测试
│   │
│   └── btc_pricing_output_v1_3/           # 估值报告输出目录 (CSV / JSON)
└── 波动率&时间序列检验/                      # 其他学术复现模块
```

---

## 4. 运行说明

在 `加密货币定价&基本面` 目录下：

### 快速诊断运行 (跳过慢速 API)
```bash
python btc_unified_pricing_model_v1_3.py --fast
```

### 自定义配置运行
```bash
python btc_unified_pricing_model_v1_3.py --config config.example.json --days 180
```

### 运行单元测试
```bash
python -m unittest discover tests
```

---

## 5. 免责声明与边界

Biais 和 Liu-Tsyvinski 模块在此项目中作为基于学术研究背景的**启发式动态折价层**实现，并非原论文计量经济学模型参数的直接校准或预测工具。
