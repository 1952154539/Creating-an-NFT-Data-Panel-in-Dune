# Azuki NFT Data Panel in Dune

Dune Analytics 仪表板，用于分析 Azuki NFT 代币的链上数据。

**Azuki NFT 合约地址**: `0xED5AF388653567Af2F388E6224dC7C4b3241C544`

**Dune Dashboard**: [Azuki NFT Data Panel](https://dune.com/) *(替换为你的仪表板链接)*

---

## 仪表板包含

| 面板 | 描述 |
|------|------|
| 持有者总数 | Azuki NFT 的独立持有地址数量 |
| 持有人名单 | 持有地址及其持有的 NFT 数量 |
| OpenSea 交易量 | 最近 30 天在 OpenSea 的交易量趋势 |

---

## 1. 持有者总数

统计当前持有 Azuki NFT 的独立地址数量。

```sql
WITH latest_holder AS (
  SELECT
    "to"   AS holder,
    tokenId,
    ROW_NUMBER() OVER (
      PARTITION BY tokenId
      ORDER BY evt_block_number DESC, evt_index DESC
    ) AS rn
  FROM erc721_ethereum.evt_Transfer
  WHERE contract_address = 0xED5AF388653567Af2F388E6224dC7C4b3241C544
)
SELECT COUNT(DISTINCT holder) AS total_holders
FROM latest_holder
WHERE rn = 1
  AND holder != 0x0000000000000000000000000000000000000000
```

> **可视化**: 选择 **Counter** 图表类型，显示单一数值。

---

## 2. 持有人名单

列出每个持有地址及其持有的 Azuki NFT 数量，按持有量降序排列。

```sql
WITH latest_holder AS (
  SELECT
    "to"   AS holder,
    tokenId,
    ROW_NUMBER() OVER (
      PARTITION BY tokenId
      ORDER BY evt_block_number DESC, evt_index DESC
    ) AS rn
  FROM erc721_ethereum.evt_Transfer
  WHERE contract_address = 0xED5AF388653567Af2F388E6224dC7C4b3241C544
)
SELECT
  holder,
  COUNT(tokenId) AS nft_count
FROM latest_holder
WHERE rn = 1
  AND holder != 0x0000000000000000000000000000000000000000
GROUP BY holder
ORDER BY nft_count DESC
```

> **可视化**: 选择 **Table** 图表类型，列名为 `Holder` 和 `NFT Count`。

---

## 3. 最近一段时间在 OpenSea 的交易量

统计最近 30 天在 OpenSea（含 Seaport 协议）上的 Azuki 交易量。

```sql
SELECT
  DATE_TRUNC('day', block_time) AS trade_date,
  COUNT(*)                      AS trades,
  SUM(amount_original)          AS volume_eth,
  AVG(amount_original)          AS avg_price_eth
FROM nft.trades
WHERE nft_contract_address = 0xED5AF388653567Af2F388E6224dC7C4b3241C544
  AND blockchain = 'ethereum'
  AND platform IN ('opensea', 'opensea v2', 'seaport')
  AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1
ORDER BY 1 DESC
```

> **可视化**: 选择 **Bar Chart** 或 **Area Chart**，X 轴为 `trade_date`，Y 轴为 `volume_eth`。

---

## 可选：更多分析查询

### 持有者分布（按持有数量分组）

```sql
WITH latest_holder AS (
  SELECT
    "to"   AS holder,
    tokenId,
    ROW_NUMBER() OVER (
      PARTITION BY tokenId
      ORDER BY evt_block_number DESC, evt_index DESC
    ) AS rn
  FROM erc721_ethereum.evt_Transfer
  WHERE contract_address = 0xED5AF388653567Af2F388E6224dC7C4b3241C544
),
holder_counts AS (
  SELECT
    holder,
    COUNT(tokenId) AS nft_count
  FROM latest_holder
  WHERE rn = 1
    AND holder != 0x0000000000000000000000000000000000000000
  GROUP BY holder
)
SELECT
  CASE
    WHEN nft_count = 1 THEN '1'
    WHEN nft_count BETWEEN 2 AND 5 THEN '2-5'
    WHEN nft_count BETWEEN 6 AND 10 THEN '6-10'
    WHEN nft_count > 10 THEN '10+'
  END AS holding_range,
  COUNT(*) AS holder_count
FROM holder_counts
GROUP BY 1
ORDER BY 1
```

> **可视化**: Pie Chart

---

### 全部市场交易量对比（OpenSea vs Blur vs LooksRare）

```sql
SELECT
  DATE_TRUNC('day', block_time) AS trade_date,
  platform,
  SUM(amount_original) AS volume_eth
FROM nft.trades
WHERE nft_contract_address = 0xED5AF388653567Af2F388E6224dC7C4b3241C544
  AND blockchain = 'ethereum'
  AND block_time >= NOW() - INTERVAL '30' DAY
GROUP BY 1, 2
ORDER BY 1 DESC, 2
```

> **可视化**: Stacked Bar Chart 或 Grouped Line Chart

---

## 在 Dune 上创建仪表板的步骤

### 第一步：注册并登录
访问 [Dune Analytics](https://dune.com/)，注册账号并登录。

### 第二步：创建查询
1. 点击顶部导航栏的 **New Query**（+ 按钮）
2. 选择 **Ethereum** 作为数据源
3. 将上面的 SQL 查询粘贴到编辑器中
4. 点击 **Run** 执行查询
5. 在结果下方选择合适的 **Visualization** 类型
6. 为查询命名（如 "Azuki - Total Holders"）
7. 点击 **Save** 保存

### 第三步：创建仪表板
1. 点击 **Create** → **New Dashboard**
2. 命名为 "Azuki NFT Data Panel"
3. 在每个已保存的查询页面中，点击 **Add to Dashboard**，选择刚创建的仪表板
4. 进入仪表板页面，拖拽调整布局
5. 添加文本说明框描述各部分

### 第四步：发布分享
点击仪表板右上角的 **Share** 按钮获取分享链接。

---

## 数据表说明

| 表名 | 数据来源 | 用途 |
|------|---------|------|
| `erc721_ethereum.evt_Transfer` | 原始以太坊 Transfer 事件 | 追踪 NFT 所有权变更 |
| `nft.trades` | Spellbook 聚合表 | 跨平台 NFT 交易数据 |

---

## 合约信息

- **项目名称**: Azuki
- **代币标准**: ERC-721
- **区块链**: Ethereum
- **合约地址**: `0xED5AF388653567Af2F388E6224dC7C4b3241C544`
- **Etherscan**: [Azuki Token Tracker](https://etherscan.io/token/0xED5AF388653567Af2F388E6224dC7C4b3241C544)
- **OpenSea**: [Azuki Collection](https://opensea.io/collection/azuki)
