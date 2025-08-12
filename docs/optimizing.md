# 优化指南

Passivbot 的配置可通过多目标进化算法进行优化，以在满足约束的前提下平衡各项性能指标。

## 启动优化

```bash
python3 src/optimize.py [path/to/config.json]
```

- 如果未指定配置文件，默认使用 `configs/template.json`
- 可通过 `--start path/to/config(s)` 指定已有配置作为起点

示例：
```bash
python3 src/optimize.py configs/template.json --start configs/starting_pool/
```

大多数参数都可通过命令行修改。更多用法请运行：`python3 src/optimize.py -h`

## 优化流程

- 采用 NSGA-II 遗传算法进化配置
- 基于历史 OHLCV 数据进行回测
- 多进程并行，使用共享内存以降低内存占用
- 持续维护表现最优的 Pareto 前沿解集
- 通过 `optimize.limits` 强制约束条件
- 通过 `optimize.scoring` 优化多个指标
- 通过哈希追踪和扰动避免重复配置

## 输出结构

每次优化会生成一个目录：
```
optimize_results/YYYY-MM-DDTHH_MM_SS_{exchanges}_{n_days}days_{coin_label}_{hash}/
```

目录内容：
- `all_results.bin`：所有评估过的配置的二进制日志（msgpack 格式）
- `pareto/`：Pareto 最优配置的 JSON 文件
  - 文件名为 `{distance}_{hash}.json`，其中 `distance` 表示到理想点的归一化距离
- `index.json`：Pareto 成员的哈希列表

## 结果分析

每个 Pareto 成员都包含完整分析。可用如下命令生成可视化图表，支持 2 或 3 个指标的绘图：
```bash
python3 src/pareto_store.py optimize_results/.../pareto/
```

## 优化约束

如需在优化过程中强制约束，可用 `optimize.limits` 键。每个约束定义一个阈值，超出后将对配置进行惩罚，惩罚强度随超标程度递增。支持命令行和配置文件两种格式。

### 命令行格式：
示例：
```bash
--limits "--penalize_if_greater_than_drawdown_worst 0.3 --penalize_if_lower_than_adg 0.001"
```

含义：
- 若 `drawdown_worst > 0.3`，则对该配置进行惩罚
- 若 `adg < 0.001`，则对该配置进行惩罚

### 配置文件格式：
```json
"limits": {
  "penalize_if_greater_than_drawdown_worst": 0.3,
  "penalize_if_lower_than_adg": 0.001
}
```

### 说明：
- 若约束键仅为指标名，则方向由其评分权重自动推断
- 指标名可为普通（如 `adg`，用于 USD 保证金回测）或加前缀 `btc_`（如 `btc_adg`，用于 BTC 保证金回测）
- 惩罚会加到目标分数上，但不会直接淘汰配置
- 惩罚强度呈指数增长，但有上限以保证稳定

## 性能指标

### 收益与增长
| Metric | 说明 |
|--------|------|
| `adg` | 平均日收益率（平滑几何平均） |
| `mdg` | 日收益中位数 |
| `gain` | 最终余额增长（期末/期初余额比） |

### 风险指标
| Metric | 说明 |
|--------|------|
| `drawdown_worst` | 最大回撤 |
| `drawdown_worst_mean_1pct` | 最差 1% 回撤的均值 |
| `expected_shortfall_1pct` | 最差 1% 日亏损均值（CVaR） |
| `equity_balance_diff_neg_max` | 资产与余额最大负偏差 |
| `equity_balance_diff_neg_mean` | 资产与余额负偏差均值 |
| `equity_balance_diff_pos_max` | 资产与余额最大正偏差 |
| `equity_balance_diff_pos_mean` | 资产与余额正偏差均值 |

### 比率与效率
| Metric | 说明 |
|--------|------|
| `sharpe_ratio` | 收益/波动率比 |
| `sortino_ratio` | 收益/下行波动率比 |
| `calmar_ratio` | 收益/最大回撤比 |
| `sterling_ratio` | 收益/平均最差 1% 回撤比 |
| `omega_ratio` | 日正收益之和 / 日负收益绝对值之和 |

### 仓位相关指标
| Metric | 说明 |
|--------|------|
| `positions_held_per_day` | 日均持仓数 |
| `position_held_hours_max` | 单笔持仓最长时间（小时） |
| `position_held_hours_mean` | 持仓平均时长（小时） |
| `position_held_hours_median` | 持仓时长中位数（小时） |
| `position_unchanged_hours_max` | 持仓未调整的最长时间（小时） |

### 权益曲线质量
| Metric | 说明 |
|--------|------|
| `equity_choppiness` | 权益曲线总波动归一化（越低越平滑） |
| `equity_jerkiness` | 权益曲线二阶导绝对值均值归一化 |
| `exponential_fit_error` | 对数线性拟合均方误差（越低增长越稳定） |

> 带 `_w` 后缀的指标为加权均值，强调近期表现。具体为：在权益曲线的 10 个重叠子区间（全区间、后 1/2、后 1/3 ... 到后 1/10）分别计算指标后取平均。

## 工具方法

以编程方式加载结果：
```python
from opt_utils import load_results

for config in load_results("optimize_results/.../all_results.bin"):
    # 处理 config
```

