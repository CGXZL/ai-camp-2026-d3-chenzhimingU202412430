# 每日作业报告

## 1. 本日问题

- 里程碑：day-03
- 学生或小组：懒得起名组（高川石、潘锐、徐梓朔、陈治名）
- 使用者：希望提前看到用电变化、但不能依赖昂贵系统的家庭能源研究小组。
- 真实输入：Kaggle Household Electric Power Consumption 的 `household_power_consumption.txt`（2,075,259 条按分钟记录）。
- 需要的输出：先运行“上一小时 ≈ 下一小时”的持续性基线，再运行同一条件下的固定随机森林候选，输出同一后段测试区间上的 MAE、最大误差时刻、高需求告警计数和真实失败案例。
- 与使用者最相关的错误：**漏掉高需求小时（假阴性）**——真正的高功率时段没有被提示；其次是误报（假阳性），会消耗使用者对提示的信任。两个模型都只识别出 38 个高需求小时中的 7 个，这是对使用者最危险的限制。
- 本日产品边界：一户历史数据不能代表其他家庭，也不能直接用于电网控制或安全告警。

## 2. 真实数据或真实课程输入

- 所有者/发布者：UCI Machine Learning Repository（Kaggle 镜像 `uciml/electric-power-consumption-data-set`）。
- 标题：Individual household electric power consumption。
- 原始 URL：https://www.kaggle.com/datasets/uciml/electric-power-consumption-data-set
- 许可标签或使用许可：按 Kaggle 页面标注的许可使用；仅用于本课程教学。
- 下载/取得日期：2026-08-18（真实文件已放入 `data/raw/household_power_consumption.txt`，未由智能体生成替代数据）。
- 预期文件与结构：`data/raw/household_power_consumption.txt`，2,075,259 条分钟记录、9 列（Date;Time;Global_active_power;Global_reactive_power;Voltage;Global_intensity;Sub_metering_1;Sub_metering_2;Sub_metering_3），分号分隔。
- 检查命令：`python analyze.py --check-data`
- 实际检查结果：

```text
REAL DATA CHECK PASSED
rows: 2075259
columns: 9
time_range: ['2006-12-16 17:24:00', '2010-11-26 21:02:00']
missing_global_active_power: 25979
```

- 已知缺失、偏差或限制：`Global_active_power` 有 25979 个缺失分钟值（占约 1.3%），课程流程在聚合前丢弃；本实验只取前 150000 条真实分钟记录（2006-12-16 至 2007-03-30）作为教室窗口；数据只覆盖单户历史用电，不代表其他家庭或实时电网状态。

## 3. 可复现运行

```powershell
# 当前目录
cd student-work\day-03-power

# 安装
python -m pip install -r requirements.txt

# 数据检查
python analyze.py --check-data
# 预期：REAL DATA CHECK PASSED；rows: 2075259；time_range 2006-12-16 17:24:00 至 2010-11-26 21:02:00

# 准备小时数据
python analyze.py --prepare
# 预期：source_rows_requested: 150000；hourly_rows: 2501

# 主程序（生成 metrics.json、largest_errors.csv、forecast.png）
python analyze.py

# 测试
python -m unittest discover -s tests -v
```

关键输出与位置：`metrics.json` 写入基线与候选指标；`largest_errors.csv` 写入误差最大的 12 个真实时刻；`forecast.png` 画留出一周对比。三者都由上述命令用真实数据、固定种子（`random_state=42`）和时间顺序划分（前 80% 训练、后 20% 测试）重复产生。

## 4. 基线与候选

### 简单基线

- 方法：持续性基线——用最近一小时（`lag_1`）的功率直接作为下一小时预测，即“下一小时和刚过去的一小时差不多”。
- 为什么足够简单：只有一个规则、不学习、可解释，是判断“任何更复杂模型是否值得”的最低参照。
- 命令：`python analyze.py`
- 结果：MAE 0.794 kW；RMSE 1.112 kW。在 3.16 kW 高需求阈值下：TP 7、FP 31、FN 31，高需求召回率 0.184。

### 候选方法

- 学生完成的核心改动：实现 `make_lagged()`（按时间排序、生成 `lag_1/2/3/24` 与 `hour_of_day`、目标 `target_next`、丢弃缺失行）和 `build_candidate()`（固定种子 `RandomForestRegressor(n_estimators=100, random_state=42)`）；并补齐课程文档承诺的 `--check-data` 真实数据检查命令。
- 保持不变的数据、划分、指标或参数：`LAGS`、`chronological_split`（前 80% / 后 20%）、MAE/RMSE 定义、高需求阈值（训练集 `target_next` 的 90 分位数 3.16 kW）、测试断言全部未改；没有让未来数据进入特征，没有随机打乱时间顺序。
- 命令：`python analyze.py`
- 结果：MAE 0.573 kW；RMSE 0.789 kW。高需求阈值下：TP 7、FP 9、FN 31，高需求召回率 0.184。

| 项目 | 基线 | 候选 | 含义 |
| --- | ---: | ---: | --- |
| 主指标（MAE） | 0.794 kW | 0.573 kW | 每个测试小时预测与真实平均功率的平均绝对距离 |
| RMSE | 1.112 kW | 0.789 kW | 对大误差更敏感的均方根误差 |
| 高需求误报 FP | 31 | 9 | 实际未达 3.16 kW 却被标记为高需求的小时数 |
| 高需求漏报 FN | 31 | 31 | 实际达到高需求却被漏掉的小时数 |
| 高需求召回率 | 0.184 | 0.184 | 38 个高需求小时中只识别出 7 个 |

在同一个 496 小时后段测试区间上，随机森林的 MAE/RMSE 明显低于基线，且误报从 31 降到 9。但两者对“真正的高需求小时”召回率完全相同（0.184）——候选在平稳时段更准，却没有更会“报尖峰”。这说明：平均误差的改善不能自动转化为更好的高需求告警，对使用者最关心的高需求时段，两个模型都还不够。

## 5. 一个真实失败案例

- 样本位置/编号：`largest_errors.csv` 第一行；`timestamp=2007-03-28 17:00:00`。
- 真实结果：下一小时平均功率 0.386 kW（非常低，接近空载）。
- 系统输出：预测 3.606 kW——比真实值高 3.22 kW，是全部测试小时中误差最大的一刻。
- 可以观察到什么：这是下午 17 点。滞后特征（`lag_1/2/3/24`）都来自过去的小时，若此前傍晚时段用电一直较高（约 3.6 kW），模型就会把“下一小时也差不多”当作默认，而真实用电在这一小时突然掉到接近零——典型的突然离家或负荷骤降。
- 说明的限制：滞后特征只能“重复过去”，无法感知即将发生的突变；模型对突增和突降都反应迟钝。第二个真实案例 `2007-03-11 12:00:00` 是反方向：真实 3.31 kW、预测 0.77 kW，模型在中午突然出现的用电尖峰上严重低估。
- 不能证明什么：不能证明“该家庭一定出门了”，也不能说“这类突变不可预测”或“换一个模型就能解决”；只能说明在本次时间窗口内，基于滞后的模型在这两个真实时刻失准。
- 下一项最小检查：把 `lag_24`（前一天的同一小时）加入比较，或检查这些高误差时刻的“此前 1–3 小时功率变化率”，看是否能在特征层面捕捉突变信号，再决定是否改进。

## 6. 智能体与学生工作边界

- 智能体提出/生成/修改了什么：按课程要求实现 `make_lagged` 与 `build_candidate` 两个 TODO；根据文档补齐 `--check-data` 命令；生成本报告的结构与运行记录。
- 学生怎样核对文件、来源、输出、测试和 diff：逐行复核 `analyze.py` 的滞后与划分代码、对比 `metrics.json`/`largest_errors.csv` 与终端输出、运行 `git diff --check`，并用 README 命令从零重跑一遍确认可复现。
- 学生修改或拒绝了什么建议：拒绝任何随机打乱时间顺序或改测试阈值的建议；保持数据、划分、指标与测试不变。
- 每名成员能独立解释的代码或证据：`make_lagged` 为什么只用过去、`chronological_split` 为什么按时间切、MAE 与召回率各说明什么、`largest_errors.csv` 中失败案例的成因。

## 7. 结论与限制

在 2006-12-16 至 2007-03-30 的前 150000 条真实分钟记录（2501 小时，前 1980 小时训练、后 496 小时测试）上，固定随机森林候选的 MAE（0.573 kW）显著低于持续性基线（0.794 kW），说明在平稳时段引入滞后特征值得额外复杂度。但**数据限制**是：只用单户、冬季四个月的窗口，不能推广到其他家庭或季节；**方法限制**是：两个模型对高需求小时的召回率都只有 0.184，且最大误差 3.22 kW 来自无法用滞后特征预见的突然负荷骤降，平均指标掩盖了这类对使用者最重要的失败；**使用边界**是：本结果不能直接用于电网控制或安全告警，也不能作为“家庭行为可预测”的证据。证据支持的最小结论是：在这份真实单户数据上，候选模型在平均误差上优于基线，但在高需求告警上两者同样不足。

## 8. 提交复核

- [x] README 从新环境可以开始运行
- [x] 数据检查、测试和主程序重新运行
- [x] 报告数字与保存输出一致（metrics.json / largest_errors.csv）
- [ ] `presentation.pptx` 在 3 分钟内讲完（见 speaking-script.md 讲稿）
- [ ] `submission.json` 路径正确
- [x] 无密钥、大数据、私人信息、虚拟环境或缓存
- [ ] GitHub 网页复查并邮件发送 URL
