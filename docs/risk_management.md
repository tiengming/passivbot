# 风险管理

## Leverage（杠杆）

在永续合约市场中，Passivbot 可能会让持仓金额超过钱包余额的 100%。  

Passivbot 只用未加杠杆的钱包余额进行风险和下单计算，因此只要交易所杠杆设置足够高，能满足机器人下单需求，实际杠杆设置不会影响风险、收益或机器人的行为。实际操作中，建议杠杆至少高于 total_wallet_exposure_limit_long + total_wallet_exposure_limit_short，这样机器人才能有足够的保证金下单，否则可能会遇到“保证金不足”错误。

## Wallet Exposure（钱包风险敞口）

衡量单个持仓风险时，Passivbot 计算持仓规模（以报价币计）与未加杠杆钱包余额的比值。  
计算公式：

`wallet_exposure = (position_size * position_price) / unleveraged_wallet_balance`  

wallet_exposure==0.0 表示无持仓。
wallet_exposure==1.0 表示持仓金额等于未加杠杆钱包余额。
wallet_exposure==4.0 表示持仓金额为未加杠杆钱包余额的 4 倍。

例如，钱包余额为 $1000，做多持仓为 100.0，持仓价格为 35.0，  
则 wallet_exposure = `100 * 35 / 1000 == 3.5`

## Wallet Exposure Limit（单仓风险上限）

wallet_exposure_limit 表示单个持仓的风险敞口上限，超过该值机器人将不再加仓。

例如，wallet_exposure_limit=0.6 时，若某持仓 wallet_exposure >= 0.6，机器人将不再补仓。

## Total Wallet Exposure Limit（总风险上限）

用户可分别为多头和空头设置 total_wallet_exposure_limit。单个持仓的 wallet_exposure_limit = total_wallet_exposure_limit / n_positions。

## Bankruptcy and liquidation（爆仓与强平）

爆仓指 `equity == (balance + unrealized_pnl) == 0.0`，即总负债等于总资产。

强平是指交易所为防止账户变成负资产而强制平仓，通常会在真正爆仓前发生，以覆盖滑点等成本。

爆仓价格可根据持仓和余额计算：

例如：  
如果 wallet_exposure==1.0，爆仓价为 0。  
如果 wallet_exposure==2.0，爆仓价比持仓价低 50%。  
如果 wallet_exposure==3.0，爆仓价比持仓价低 33.33%。  
如果 wallet_exposure==10.0，爆仓价比持仓价低 10%。  


## Getting stuck（被套）

当机器人无法再加仓且 wallet_exposure_limit 已达到或超过时，称为“被套”。  

如果多头被套且价格持续下跌，持仓价与市价的距离会越来越大，想要小幅反弹就平仓获利的可能性也会降低。此时，若想让持仓价更接近市价，需继续加仓，但加仓越多，若行情继续不利，爆仓风险也越大。

## Diversification（分散化）

虽然大多数市场（尤其是加密市场）普遍存在相关性（如比特币暴跌时其他币也会跟跌），但不同币种的反弹时机和幅度往往不同。因此，分散持仓于多个币种有助于降低单一币种爆仓导致全局亏损的风险。

比如抛硬币 1000 次，正反面大致各 500 次；但只抛一次，要么正面要么反面。所以，与其让 1 个机器人被套 wallet_exposure==1.0，不如让 10 个机器人各被套 wallet_exposure==0.1，风险更分散。

