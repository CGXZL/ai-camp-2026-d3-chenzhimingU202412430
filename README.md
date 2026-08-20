# D3 预测下一小时家庭用电（AI 工程营）

根据过去的家庭用电测量，比较“上一小时 ≈ 下一小时”的持续性基线与一个固定随机森林候选模型，在同一后段测试区间公平比较，预测下一小时平均功率，并诚实解释真实失败时刻。

- **使用者**：希望提前看到用电变化、但不能依赖昂贵系统的家庭能源研究小组。
- **真实输入**：Kaggle Household Electric Power Consumption 的 `household_power_consumption.txt`（2,075,259 条按分钟记录）。
- **需要的输出**：先运行持续性基线，再运行同一条件下的随机森林候选，输出 MAE、最大误差时刻、高需求告警计数与真实失败案例。
- **本日产品边界**：一户历史数据不能代表其他家庭，也不能直接用于电网控制或安全告警。

## 项目结构

```text
.
├── data/raw/household_power_consumption.txt  # 真实 Kaggle 分钟数据（不入库）
├── data/processed/hourly_power.csv           # 运行生成：小时平均功率
├── analyze.py            # 主程序：数据检查、聚合、滞后特征、基线、候选、评估
├── tests/test_sequence.py
├── requirements.txt
├── report.md             # 书面报告
├── presentation.pptx     # 3 分钟答辩
├── speaking-script.md    # 答辩讲稿
├── submission.json       # 提交清单
├── metrics.json          # 运行生成：基线与候选指标
├── largest_errors.csv    # 运行生成：误差最大的真实时刻
└── forecast.png          # 运行生成：留出一周预测对比图
```

## 环境

- Python 3.11+（本机验证：Python 3.14）
- 依赖见 `requirements.txt`（`numpy>=2,<3`、`pandas>=2.2,<4`、`scikit-learn>=1.5,<2`、`matplotlib>=3.9`）

## 安装

```powershell
python -m pip install -r requirements.txt
```

## 数据

从指定页面下载：

https://www.kaggle.com/datasets/uciml/electric-power-consumption-data-set

把下载解压后的 `household_power_consumption.txt` 放到 `data/raw/household_power_consumption.txt`。不改文件名、不生成替代数据、不提交原始大数据（`data/raw/` 与 `data/processed/` 已被 `.gitignore` 忽略）。

## 数据检查

```powershell
python analyze.py --check-data
```

预期输出：

```text
REAL DATA CHECK PASSED
rows: 2075259
columns: 9
time_range: ['2006-12-16 17:24:00', '2010-11-26 21:02:00']
missing_global_active_power: 25979
```

若失败，按顺序检查：当前目录、`data/raw/household_power_consumption.txt` 是否存在、解压层级、下载来源；不要修改检查器预期。

## 准备小时数据

```powershell
python analyze.py --prepare
```

读取前 `150000` 条真实分钟记录，聚合为小时平均功率，写入 `data/processed/hourly_power.csv`（预期约 2501 小时，2006-12-16 17:00 至 2007-03-30 21:00）。

## 运行主程序

```powershell
python analyze.py
```

- 输出指标写入 `metrics.json`；
- 输出误差最大的 12 个真实时刻写入 `largest_errors.csv`；
- 输出留出一周对比图 `forecast.png`；
- 数据划分：按时间先后，前 80% 训练、后 20% 测试（`chronological_split`）；
- 特征只来自预测时刻之前的滞后值 `lag_1, lag_2, lag_3, lag_24` 与 `hour_of_day`；
- 基线：用最近一小时功率预测下一小时；
- 候选模型固定种子：`RandomForestRegressor(n_estimators=100, random_state=42)`。

## 运行测试

```powershell
python -m unittest discover -s tests -v
```

## 限制

- 只使用单户历史数据，结果不能推广到其他家庭；
- 模型预测不能直接用于电网控制或安全告警；
- 本数据包含 25979 个缺失的 `Global_active_power` 分钟值，已按课程流程丢弃后聚合。
