# Passivbot 参数说明

本文档概述了 `config/template.json` 中各参数的含义。

## 回测设置

- **base_dir**：保存回测结果的位置。
- **compress_cache**：设为 `true` 可节省磁盘空间，设为 `false` 可加快加载速度。
- **end_date**：回测结束日期，例如 `2024-06-23`。设为 `'now'` 表示使用今天作为结束日期。
- **exchanges**：用于回测和优化时获取 1 分钟 OHLCV 数据的交易所。可选项：[binance, bybit, gateio, bitget]。
- **start_date**：回测开始日期。
- **starting_balance**：回测开始时的初始 USD 余额。
- **use_btc_collateral**：`true`/`false`。设为 `true` 时，回测以 BTC 作为保证金，模拟初始为 100% BTC，所有 USD 盈利买入 BTC，亏损时不卖出 BTC（而是形成 USD 负债）。
  - 例：BTC/USD 价格为 `$100,000`，BTC 余额为 `1.0`，回测盈利 `$10`，BTC 余额变为 `1.0001`，USD 余额为 `0`。若亏损 `$20`，BTC 余额仍为 `1.0001`，USD 余额为 `-20`。若再盈利 `$15`，先偿还 USD 负债：BTC 余额不变，USD 余额为 `-5`。若再盈利 `$10`，BTC 余额变为 `1.00015`，USD 余额为 `0`。

## 机器人设置

### 多空通用参数

- **ema_span_0**, **ema_span_1** ：
  - EMA 跨度，以分钟为单位。
  - 公式：`next_EMA = prev_EMA * (1 - alpha) + new_val * alpha`，其中 `alpha = 2 / (span + 1)`。
  - 还会计算一个额外的 EMA 跨度：`(ema_span_0 * ema_span_1)**0.5`。
  - 三个 EMA 形成上下边界：
    - `ema_band_lower = min(emas)`
    - `ema_band_upper = max(emas)`
  - 这些边界用于初始开仓和自动解套平仓。
- **n_positions**：最大可开仓位数。设为 `0` 禁用多/空。
- **total_wallet_exposure_limit**：最大可用仓位比例。
  - 例：`total_wallet_exposure_limit = 0.75` 表示最多使用 75%（未加杠杆）钱包余额。
  - 例：`total_wallet_exposure_limit = 1.6` 表示最多使用 160%（未加杠杆）钱包余额。
  - 每个仓位分配等额份额：`wallet_exposure_limit = total_wallet_exposure_limit / n_positions`。
  - 详见：`docs/risk_management.md`。
- **enforce_exposure_limit**：若为 `true`，则对每个仓位强制执行仓位限制。
  - 例：若某仓位超出限制 1%，则以市价减仓至限制范围内。
  - 有助于风险管理，例如用户提取余额或更改设置时。

### 网格开仓参数

Passivbot 可配置为创建网格开仓订单，价格和数量由以下参数决定：

- **entry_grid_double_down_factor** ：
  - 下一个网格开仓的数量为当前仓位大小乘以该因子。
  - 例：仓位大小为 `1.4`，`double_down_factor` 为 `0.9`，则下一个开仓数量为 `1.4 * 0.9 = 1.26`。
  - 也适用于追踪开仓。
- **entry_grid_spacing_pct**, **entry_grid_spacing_weight** ：
  - 网格补仓价格的计算方式如下：
    - `next_reentry_price_long = pos_price * (1 - entry_grid_spacing_pct * modifier)`
    - `next_reentry_price_short = pos_price * (1 + entry_grid_spacing_pct * modifier)`
  - 其中 `modifier = (1 + ratio * entry_grid_spacing_weight)`，`ratio = wallet_exposure / wallet_exposure_limit`。
- **entry_initial_ema_dist** ：
  - 相对于 EMA 下/上边界的偏移量。
  - 多头初始开仓/空头解套平仓价格为下边界减去偏移。
  - 空头初始开仓/多头解套平仓价格为上边界加上偏移。
  - 详见 `ema_span_0`/`ema_span_1`。
- **entry_initial_qty_pct** ：
  - `initial_entry_cost = balance * wallet_exposure_limit * initial_qty_pct`

### 追踪参数

追踪开仓和追踪平仓逻辑相同。

- **trailing_grid_ratio** ：
  - 设置追踪与网格分配比例。
  - 若 `trailing_grid_ratio = 0.0`，仅使用网格订单。
  - 若 `trailing_grid_ratio = 1.0` 或 `-1.0`，仅使用追踪订单。
  - 若 `trailing_grid_ratio > 0.0`，先追踪后网格。
  - 若 `trailing_grid_ratio < 0.0`，先网格后追踪。
    - 例：`trailing_grid_ratio = 0.3`，仓位达到 30% 前用追踪，之后用网格。
    - 例：`trailing_grid_ratio = -0.9`，仓位达到 10% 前用网格，之后用追踪。
    - 例：`trailing_grid_ratio = -0.12`，仓位达到 88% 前用网格，之后用追踪。
- **trailing_retracement_pct**, **trailing_threshold_pct** ：
  - 触发追踪订单有两个条件：1）阈值，2）回撤。
  - 若 `trailing_threshold_pct <= 0.0`，阈值条件始终成立。
  - 多头：
    - 若自持仓以来最高价 > 持仓价 * (1 + trailing_threshold_pct)，第一个条件成立。
    - 若自最高价以来最低价 < 最高价 * (1 - trailing_retracement_pct)，第二个条件成立，触发下单。
  - Passivbot 自行追踪价格，不使用交易所的特殊追踪订单类型。
  - 每次仓位变化（加仓或部分平仓）时，追踪价格重置。
  - 追踪基于 1 分钟 OHLCV，每分钟更新一次。

### 网格平仓参数

- **close_grid_markup_start**, **close_grid_markup_end**, **close_grid_qty_pct** ：
  - 止盈（TP）价格在以下区间线性分布：
    - 多头：`pos_price * (1 + markup_start)` 到 `pos_price * (1 + markup_end)`。
    - 空头：`pos_price * (1 - markup_start)` 到 `pos_price * (1 - markup_end)`。
  - TP 方向取决于 `markup_start` 和 `markup_end` 的大小关系：
    - 若 `markup_start > markup_end`：TP 网格为反向（多头从高到低，空头从低到高）。
    - 若 `markup_start < markup_end`：TP 网格为正向（多头从低到高，空头从高到低）。
  - 例（多头，反向 TP）：`pos_price = 100`，`markup_start = 0.01`，`markup_end = 0.005`，`close_grid_qty_pct = 0.2`，TP 价格为：[101.0, 100.9, 100.8, 100.7, 100.6]。
  - 例（多头，正向 TP）：`markup_start = 0.005`，`markup_end = 0.01`，TP 价格为：[100.5, 100.6, 100.7, 100.8, 100.9]。
  - 例（空头，正向 TP）：`pos_price = 100`，`markup_start = 0.005`，`markup_end = 0.01`，TP 价格为：[99.5, 99.4, 99.3, 99.2, 99.1]。
  - 例（空头，反向 TP）：`markup_start = 0.01`，`markup_end = 0.005`，TP 价格为：[99.0, 99.1, 99.2, 99.3, 99.4]。
  - 每单数量为 `full pos size * close_grid_qty_pct`。
  - 注：full pos size 指最大仓位，若实际仓位较小，TP 订单数会少于 `1 / close_grid_qty_pct`。
  - TP 网格按 `markup_start` 到 `markup_end` 顺序分配，每单分配对应数量：
    - 第一单分配 `close_grid_qty_pct * full_pos_size`。
    - 第二单分配 `close_grid_qty_pct` 到 `2 * close_grid_qty_pct`，以此类推。
  - 例：`full_pos_size = 100`，`long_pos_size = 55`，反向 TP，TP 订单为：[15@100.8, 20@100.9, 20@101.0]。
  - 若仓位超出最大仓位，超出部分加到最靠近 `markup_start` 的 TP 单。
    - 例：`long_pos_size = 130`，正向 TP，TP 订单为：[50@100.5, 20@100.6, 20@100.7, 20@100.8, 20@100.9]。

### 追踪平仓参数

- **close_trailing_grid_ratio**：见上文追踪参数。
- **close_trailing_qty_pct**：平仓数量为 `full pos size * close_trailing_qty_pct`。
- **close_trailing_retracement_pct**：见上文追踪参数。
- **close_trailing_threshold_pct**：见上文追踪参数。

### 解套参数

若仓位被套，机器人会用其他仓位的盈利来实现被套仓位的亏损。若有多个被套仓位，优先解套价格距离最小的仓位。

- **unstuck_close_pct** ：
  - 每次解套下单的比例为 `full pos size * wallet_exposure_limit`。
- **unstuck_ema_dist** ：
  - 距离 EMA 边界的距离：
    - 多头解套平仓价 = 上 EMA 边界 * (1 + unstuck_ema_dist)
    - 空头解套平仓价 = 下 EMA 边界 * (1 - unstuck_ema_dist)
- **unstuck_loss_allowance_pct** ：
  - 允许亏损的加权百分比，相对于历史峰值余额。
  - `loss_allowance = past_peak_balance * (1 - unstuck_loss_allowance_pct * total_wallet_exposure_limit)`
  - 例：历史峰值余额为 `$10,000`，`unstuck_loss_allowance_pct = 0.02`，`total_wallet_exposure_limit = 1.5`，则机器人在余额降至 `$10,000 * (1 - 0.02 * 1.5) = $9,700` 时停止亏损。
- **unstuck_threshold** ：
  - 若仓位大于该阈值，则视为被套并启动解套。
  - `if wallet_exposure / wallet_exposure_limit > unstuck_threshold: 启动解套`
  - 例：仓位为 `$500`，最大允许仓位为 `$1000`，仓位已满 50%。若 `unstuck_threshold = 0.45`，则解套至 `$450`。

### 筛选参数

交易币种通过成交量和噪声度筛选。先按成交量过滤，剔除最低成交量的币种，再按噪声度排序，选取噪声度最高的币种进行交易。

- **filter_volume_drop_pct**：成交量过滤。剔除相对成交量最低的币种。
  - 例：`filter_volume_drop_pct = 0.1`，剔除成交量最低的 10% 币种。设为 `0` 表示全部允许。
- **filter_noisiness_rolling_window/filter_volume_rolling_window**：用于动态选币的成交量和噪声度回看分钟数。
  - 噪声度为 1 分钟 OHLCV 的归一化振幅均值：`mean((high - low) / close)`。
  - forager 模式下，机器人选取噪声度最高的币种开仓。

## Coin Overrides
- **coin_overrides**:
  - Specify full or partial configs for individual coins, overriding values from master config.
  - Format: {"COIN1": overrides1, "COIN2": overrides2}
  - Whole configs may be loaded with parameter "override_config_path". May either be full path to config, or filename for alternate config file from the same directory as master config file.
  - Specific override parameters take precedence over override parameters loaded from external config.
  - Only a subset of config parameters are eligible for overriding master config:
    - config.bot.long/short:
      ```
      [
        close_grid_markup_end, close_grid_markup_start, close_grid_qty_pct, close_trailing_grid_ratio, close_trailing_qty_pct,
        close_trailing_retracement_pct, close_trailing_threshold_pct, ema_span_0, ema_span_1, enforce_exposure_limit,
        entry_grid_double_down_factor, entry_grid_spacing_pct, entry_grid_spacing_weight, entry_initial_ema_dist,
        entry_initial_qty_pct, entry_trailing_double_down_factor, entry_trailing_grid_ratio, entry_trailing_retracement_pct,
        entry_trailing_threshold_pct, unstuck_close_pct, unstuck_ema_dist, unstuck_threshold, wallet_exposure_limit
      ]
      ```
    -config.live:
    ```
    [forced_mode_long, forced_mode_short, leverage]
    ```
  - Examples:
    - `{"COIN1": {"override_config_path": "path/to/override_config.json"}}` -- Will attempt to load "path/to/override_config.json" and apply all eligible parameters from there for COIN1
    - `{"COIN2": {"override_config_path": "path/to/other_override_config.json", {"bot": {"long": {"close_grid_markup_start": 0.005}}}}}` -- Will attempt to load `"path/to/other_override_config.json"` first, and apply `{"bot": {"long": {"close_grid_markup_start": 0.005}}}` after.
    - `{"COIN3": {"bot": {"short": {"entry_initial_qty_pct": 0.01}}, "live": {"forced_mode_long": "panic"}}}` -- Will apply given overrides for COIN3.
- **forced_modes**:
  - Choices: `[n (normal), m (manual), gs (graceful_stop), t (tp_only), p (panic)]`.
    - **Normal mode**: Passivbot manages the position as normal.
    - **Manual mode**: Passivbot ignores the position.
    - **Graceful stop**: If there is a position, Passivbot manages it; otherwise, no new positions are opened.
    - **Take Profit Only mode**: Passivbot only manages closing orders.
    - **Panic mode**: Passivbot closes the position immediately.

## Live Trading Settings

- **approved_coins**:
  - List of coins approved for trading. If empty, see `live.empty_means_all_approved`.
    - Backtester and optimizer use `live.approved_coins` minus `live.ignored_coins`.
  - May be given as a path to an external file, read continuously by Passivbot.
  - May be split into long and short:
    - Example: `{"long": ["COIN1", "COIN2"], "short": ["COIN2", "COIN3"]}`
- **auto_gs**: Automatically enable graceful stop for positions on disapproved coins.
  - Graceful stop: The bot continues trading as normal but does not open a new position after the current position is fully closed.
  - If `auto_gs=false`, positions on disapproved coins are put on manual mode.
- **empty_means_all_approved**:
  - If `true`, `approved_coins=[]` means all coins are approved.
  - If `false`, `approved_coins=[]` means no coins are approved.
- **execution_delay_seconds**: Wait `x` seconds after executing to exchange.
- **filter_by_min_effective_cost**: If `true`, disallows coins where `balance * WE_limit * initial_qty_pct < min_effective_cost`.
  - Example: If the exchange's effective minimum cost for a coin is `$5`, but the bot wants to make an order of `$2`, disallow that coin.
- **forced_mode_long**, **forced_mode_short**: Force all coins long/short to a given mode.
  - Choices: `[m (manual), gs (graceful_stop), p (panic), t (take_profit_only)]`.
- **ignored_coins**:
  - List of coins the bot will not make positions on. If there are positions on that coin, enable graceful stop or manual mode.
  - May be given as a path to an external file, read continuously by Passivbot.
  - May be split into long and short:
    - Example: `{"long": ["COIN1", "COIN2"], "short": ["COIN2", "COIN3"]}`
- **leverage**: Leverage set on the exchange. Default is `10`.
- **market_orders_allowed**: If `true`, allows Passivbot to place market orders when the order price is very close to the current market price. If `false`, only places limit orders. Default is `true`.
- **max_n_cancellations_per_batch**: Cancels `n` open orders per execution.
- **max_n_creations_per_batch**: Creates `n` new orders per execution.
- **max_n_restarts_per_day**: If the bot crashes, restart up to `n` times per day before stopping completely.
- **mimic_backtest_1m_delay**: If `true`, the bot will only update and evaluate open orders once per full minute, synchronized to the clock (e.g., 12:01:00, 12:02:00, etc.). This mimics the backtester's timestep logic and avoids intraminute updates. Useful for achieving higher fidelity between backtest and live performance.
- **minimum_coin_age_days**: Disallows coins younger than a given number of days.
- **ohlcvs_1m_rolling_window_days**: How many days of OHLCVs the bot keeps in memory. Reduce if RAM consumption is an issue.
- **ohlcvs_1m_update_after_minutes**: How many minutes old OHLCVs for a coin may be before fetching fresh ones from the exchange. Increase if rate limiting is an issue.
- **pnls_max_lookback_days**: How far into the past to fetch PnL history.
- **price_distance_threshold**: Minimum distance to current price action required for EMA-based limit orders.
- **time_in_force**: Default is Good-Till-Cancelled.
- **user**: Fetch API key/secret from `api-keys.json`.

## Optimization Settings

### Bounds

When optimizing, parameter values are within the lower and upper bounds.

### Other Optimization Parameters

- **compress_results_file**：若为 `true`，优化结果文件压缩保存以节省空间。
- **enable_overrides**：启用自定义优化器覆盖项列表。使用 `optimizer_overrides.py` 进行自定义。默认为空。
- **crossover_probability**：遗传算法中交叉概率，决定父代交换基因信息生成子代的频率。
- **iters**：每次优化会运行的回测次数。
- **mutation_probability**：遗传算法中变异概率，决定引入随机变化以保持多样性的频率。
- **n_cpus**：并行使用的 CPU 核心数。
- **population_size**：遗传优化算法的种群规模。
- **scoring** ：
  - 优化器采用两个目标，寻找帕累托前沿。
  - 根据到理想点的欧氏距离选择最优候选。
  - 默认值为日收益中位数和夏普比率。
  - 使用 NSGA-II 算法（非支配排序遗传算法 II）进行多目标优化。
  - 适应度函数内部将目标转为负值，双目标均最小化。
  - 可选项完整列表：[adg, adg_w, calmar_ratio, calmar_ratio_w, drawdown_worst, drawdown_worst_mean_1pct, equity_balance_diff_neg_max, equity_balance_diff_neg_mean, equity_balance_diff_pos_max, equity_balance_diff_pos_mean, expected_shortfall_1pct, gain, loss_profit_ratio, loss_profit_ratio_w, mdg, mdg_w, omega_ratio, omega_ratio_w, position_held_hours_max, position_held_hours_mean, position_held_hours_median, position_unchanged_hours_max, positions_held_per_day, sharpe_ratio, sharpe_ratio_w, sortino_ratio, sortino_ratio_w, sterling_ratio, sterling_ratio_w]
  - 后缀 `_w` 表示在 10 个时间子集（整体、后半、后三分之一、...、后十分之一）上取均值，对近期数据加权。
  - 例：`["mdg", "sharpe_ratio", "loss_profit_ratio"]`，`["adg", "sortino_ratio", "drawdown_worst"]`，`["sortino_ratio", "omega_ratio", "adg_w", "position_unchanged_hours_max"]`
    - 注：若 config.backtest.use_btc_collateral=True，需加前缀 "btc_"，如 btc_adg 或 btc_drawdown_worst。

### 优化限制

优化器会对超出或低于指定阈值的回测指标进行惩罚。惩罚项会加到适应度分数上，以避免不理想的配置，但不会直接淘汰。

#### 格式

可在配置文件 `optimize.limits` 下设置，也可通过 CLI `--limits` 传参。

##### 配置示例

```json
"limits": {
  "penalize_if_greater_than_drawdown_worst": 0.3,
  "penalize_if_lower_than_adg": 0.001
}
```

##### CLI arg Example:

```
python3 src/optimize.py --limits "--penalize_if_lower_than_btc_omega_ratio 3.0 --penalize_if_greater_than_loss_profit_ratio 0.5"
```