# 工具说明

## 从优化结果中提取 Pareto 前沿和最佳配置

提取出的 pareto 前沿和最佳配置会保存在 `optimize_results_analysis/` 目录。优化会话的原始结果通常在 `optimize_results/` 目录下。

```shell
python3 src/tools/extract_best_config.py path/to/all_results.txt
```

## 复制 ohlcv 数据到新目录

在 Passivbot v7.2.13 版本中，ohlcv 数据的存储位置发生了变化。运行此脚本可将旧版本已下载的数据迁移到新位置。

```shell
python3 src/tools/copy_ohlcvs_from_v7.2.12.py
```

## 根据市值生成可交易币种列表

```shell
python3 src/tools/generate_mcap_list.py
```

运行 `python3 src/tools/generate_mcap_list.py -h` 可查看参数说明：
```
  --n_coins N_COINS, -n N_COINS
                        最大市值币种数量，默认值为 100
  --minimum_market_cap_dollars MINIMUM_MARKET_CAP_MILLIONS, -m MINIMUM_MARKET_CAP_MILLIONS
                        最小市值（百万美元），默认值为 300.0
  --exchange EXCHANGE, -e EXCHANGE
                        可选：按交易所筛选币种，多个用逗号分隔，默认 None
  --output OUTPUT, -o OUTPUT
                        可选：输出路径，默认 configs/approved_coins_{n_coins}_{min_mcap}.json
```
