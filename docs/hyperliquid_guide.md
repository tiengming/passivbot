## Passivbot 在 Hyperliquid 上的快速指南

### Hyperliquid 账户创建

1. 访问 https://app.hyperliquid.xyz/，通过连接第三方钱包（更安全）或使用邮箱（较不安全）创建账户。
2. 点击“Deposit”按钮，通过 Arbitrum 网络将 USDC（不是 USDT、USDC.E 或其他稳定币）转入你的账户地址。
3. 进入“More” -> “API”，按照指引创建 API 钱包（代理钱包）。

### Passivbot 设置

1. 如果尚未安装 Passivbot，请先安装；如已安装，拉取最新 master 分支：`git pull`。
2. 更新依赖：
   ```bash
   pip install -r requirements.txt
   ```
3. 在 `api-keys.json` 中添加你的 Hyperliquid 钱包地址和 API 钱包信息：
```json
"hyperliquid_01": {
    "exchange": "hyperliquid",
    "wallet_address": "YOUR_USDC_PUBLIC_WALLET_ADDRESS",
    "private_key": "API_PRIVATE_KEY",
    "is_vault": false
}
```
`"private_key"` 填写你在 Hyperliquid API 部分创建的 API 钱包私钥。

现在可以像平常一样运行 Passivbot。注意 Hyperliquid 的最小下单金额为 10 美元：  
`initial_entry_cost = balance * (total_wallet_exposure_limit / n_positions) * initial_qty_pct`

#### 使用 Vault（类似跟单）
1. 在 HyperLiquid 顶部菜单进入“Vaults”，创建一个新的 vault。
2. 设置名称和描述（最多 250 个字符）。
3. 要查找 vault 的钱包地址，再次进入“Vaults”，在公开 vault 列表中找到你的 vault。
4. 点击你的 vault，获取其公开地址。此地址即为 `api-keys.json` 中的 `YOUR_VAULT_PUBLIC_ADDRESS`。记得将 `"is_vault"` 设为 true。

更新 `api-keys.json`：
```json
"hyperliquid_01": {
    "exchange": "hyperliquid",
    "wallet_address": "YOUR_VAULT_PUBLIC_ADDRESS",
    "private_key": "API_PRIVATE_KEY",
    "is_vault": true
}
```

更多细节请参考 Hyperliquid 官方文档。
