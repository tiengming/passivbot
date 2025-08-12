![Passivbot](docs/images/pbot_logo_full.svg)

# 支持 Bybit、OKX、Bitget、GateIO、Binance 和 Hyperliquid 的量化交易机器人

:warning: **使用风险自负** :warning:

v7.3.17


## 概述

Passivbot 是用 Python 和 Rust 编写的加密货币量化交易机器人，旨在让用户尽量少干预。  

它专为永续合约市场设计，能自动为用户挂单和撤单（限价买入/卖出）。它不预测未来价格，不用技术指标，也不追随趋势。相反，它是一种逆势做市策略，在价格上下波动时持续提供流动性，起到市场“减震器”的作用。  

Passivbot 可在历史数据上回测策略表现，内置回测器的核心计算用 Rust 编写以提升速度。同时还内置优化器，可通过成千上万次回测自动寻找更优配置，采用进化算法收敛到最优解。  

## 策略说明

Passivbot 的策略灵感来自马丁格尔加仓法。机器人会先小仓位开仓，若行情不利则多次加仓拉近持仓均价。所有订单以网格方式分布，能有效吸收突发行情。每次补仓后，机器人会快速调整止盈单，只要行情有小幅反弹（即“bounce”），就能获利平仓并重新开始。  

### 追踪订单
除了网格开/平仓，Passivbot 还可配置追踪开仓和追踪平仓。

追踪开仓时，机器人会等待价格突破某阈值后回撤一定比例再补仓。追踪平仓时，机器人会等价格有利波动后回撤一定比例再挂止盈单。这样有助于在市场反转信号出现时更有效地锁定利润，而不是仅按固定距离止盈。

网格和追踪订单可灵活组合，既可全部用网格，也可全部用追踪，或部分用网格、部分用追踪。

### Forager
Forager 功能会动态选择近期波动性最大的市场开仓。波动性定义为最近 1 分钟 K 线的归一化振幅均值，即 `mean((ohlcv.high - ohlcv.low) / ohlcv.close)`。

### 解套机制
Passivbot 会主动管理表现不佳（“被套”）的仓位，逐步实现小额亏损。如果有多个被套仓位，优先解套持仓价与现价距离最小的仓位。机器人会限制亏损，确保账户余额不会低于历史峰值的一定比例。  

## 安装方法

按照以下步骤安装 Passivbot 及其依赖。

### 步骤 1：克隆仓库

先将 Passivbot 仓库克隆到本地：

```sh
git clone https://github.com/enarjord/passivbot.git
cd passivbot
```


### 步骤 2：安装 Rust
Passivbot 部分功能依赖 Rust。请访问 https://www.rust-lang.org/tools/install 按指引安装 Rustup（Rust 安装和版本管理工具）。安装完成后重启终端。

### 步骤 3：创建并激活虚拟环境

建议使用虚拟环境管理依赖：

```sh
python3 -m venv venv
```

激活虚拟环境：

```sh
source venv/bin/activate
```

### 步骤 4：安装 Python 依赖

安装 `requirements.txt` 中所有依赖：

```sh
pip install -r requirements.txt
```

### 步骤 5（可选）：手动编译 Rust 扩展

Passivbot 启动时会自动尝试编译 Rust 扩展，也可手动编译：

```sh
cd passivbot-rust
maturin develop --release
cd ..
```

如 Rust 源码有变动需重新编译，Passivbot 启动时会自动检测并尝试编译，也可手动执行上述命令。

### 步骤 6：添加 API 密钥

复制 api-keys 模板文件：

```sh
cp api-keys.json.example api-keys.json
```

将你的密钥填入 api-keys.json。

### 步骤 7：运行 Passivbot

用默认配置启动机器人：

```sh
python3 src/main.py -u {account_name_from_api-keys.json}
```

或用自定义配置文件（以 `configs/template.json` 为模板）：


```sh
python3 src/main.py path/to/config.json
```

## Jupyter Lab

Jupyter lab 需在与机器人相同的虚拟环境下运行。激活 venv（见上文步骤 3），并在 Passivbot 根目录启动 Jupyter lab:
```shell
python3 -m jupyter lab
```

## 运行环境要求

- Python >= 3.8
- 依赖见 [requirements.txt](requirements.txt)

## 预优化配置

即将上线...

更多配置见 https://pbconfigdb.scud.dedyn.io/

## 文档

更多详细说明见 [docs/](docs/) 目录。

## 社区支持

[![Discord](https://img.shields.io/badge/Discord-7289DA?style=for-the-badge&logo=discord&logoColor=white)](https://discord.gg/QAF2H2UmzZ)

[![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/passivbot_futures)

## 第三方服务、推荐与打赏

**Passivbot Manager Service:**  
付费托管服务，帮你运行 Passivbot：  
www.passivbotmanager.com  

**Passivbot GUI**
Passivbot 图形界面：  
https://github.com/msei99/pbgui

**推荐注册链接：**  
欢迎使用以下推荐链接支持项目：  
https://accounts.binance.com/register?ref=TII4B07C  
https://partner.bybit.com/b/passivbot  
https://partner.bitget.com/bg/Y8FU1W  
https://www.okx.com/join/PASSIVBOT  (20% 返佣)  
https://app.hyperliquid.xyz/join/PASSIVBOT  
https://app.defx.com/join/4DKR14  

**关于 Binance**  
为支持 Passivbot 持续开发，请使用满足以下条件的 Binance 账户：  
1) 2024-09-21 之后注册；  
2) 并且：  
  a) 注册时未填写推荐码，或  
  b) 注册时填写推荐码："TII4B07C"。

只有满足上述条件的账户，Passivbot 才能获得交易返佣。


**BuyMeACoffee:**  
https://www.buymeacoffee.com/enarjord  

**捐赠：**  
如果机器人为你带来盈利，欢迎捐赠支持开发：  

- USDT 或 USDC Binance Smart Chain BEP20:  
0x4b7b5bf6bea228052b775c052843fde1c63ec530  
- USDT 或 USDC Arbitrum One:  
0x4b7b5bf6bea228052b775c052843fde1c63ec530  

比特币（BTC）通过 Strike：  
enarjord@strike.me

## License
这是发布到公共领域的免费且无限制的软件。

任何人都可以免费复制，修改，发布，使用，编译，出售或以源代码形式或编译为二进制，出于任何目的，商业或非商业，以及任何目的方法。

在承认版权法的司法管辖区中，作者或作者该软件专用于任何版权通往公共领域的软件。我们为利益做出了这种奉献精神全面的公众，并损害了我们的继承人和继任者。我们打算将这种奉献精神成为公开的行为永久保留所有现在和未来的权利软件根据版权法。

该软件是“原样”提供的，没有任何形式的保证，明示或暗示，包括但不限于适销性，适合特定目的的适合性和非侵权。在任何情况下，作者都不应对任何索赔，损害赔偿或其他责任，无论是在合同，侵权还是其他责任中，是由软件或使用或使用或与使用或相关的软件中的其他交易。
For more information, please refer to <https://unlicense.org/>
