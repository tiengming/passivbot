# 回测说明

Passivbot 内置回测器，可在历史价格数据上模拟机器人的实际表现。所有币种的历史 1 分钟 K 线数据会自动下载并缓存。

## 使用方法

```shell
python3 src/backtest.py
```
或
```shell
python3 src/backtest.py path/to/config.json
```
如果未指定配置文件，默认使用 `configs/template.json`

## 回测结果

各项指标和图表会输出到 `backtests/{exchange}/` 目录。

## 回测命令行参数

- `-dp` 禁用单币种绘图。
- `-co` 将多个交易所的 ohlcv 数据合并为一个数组，否则分别回测每个交易所。

如需完整参数列表：
```shell
python3 src/backtest.py -h
```

