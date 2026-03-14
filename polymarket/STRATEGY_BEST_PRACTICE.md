# Polymarket 交易策略最佳实践

> 基于 GitHub 开源项目研究、实战经验总结和社区最佳实践
> 更新时间: 2026-03-14 12:00 UTC

---

## 🆕 最新更新 (2026-03-14)

### py-clob-client 官方 SDK 使用指南

**安装**:
```bash
pip install py-clob-client
```

**核心操作**:

```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import OrderArgs, OrderType, MarketOrderArgs
from py_clob_client.order_builder.constants import BUY, SELL

# 初始化客户端
HOST = "https://clob.polymarket.com"
CHAIN_ID = 137  # Polygon

# EOA/MetaMask 钱包
client = ClobClient(HOST, key=PRIVATE_KEY, chain_id=CHAIN_ID, signature_type=0)
client.set_api_creds(client.create_or_derive_api_creds())

# Email/Magic 钱包（需要 funder 地址）
client = ClobClient(HOST, key=PRIVATE_KEY, chain_id=CHAIN_ID, 
                    signature_type=1, funder=FUNDER_ADDRESS)
client.set_api_creds(client.create_or_derive_api_creds())

# 获取市场数据
midpoint = client.get_midpoint(token_id)
price = client.get_price(token_id, side="BUY")
orderbook = client.get_order_book(token_id)

# 市价单（按金额买入）
mo = MarketOrderArgs(token_id=token_id, amount=25.0, side=BUY, order_type=OrderType.FOK)
signed = client.create_market_order(mo)
resp = client.post_order(signed, OrderType.FOK)

# 限价单
order = OrderArgs(token_id=token_id, price=0.50, size=10.0, side=BUY)
signed = client.create_order(order)
resp = client.post_order(signed, OrderType.GTC)

# 管理订单
open_orders = client.get_orders(OpenOrderParams())
client.cancel(order_id)
client.cancel_all()
```

**⚠️ Token Allowances (MetaMask/EOA 用户必读)**:

使用 MetaMask 或硬件钱包前，必须设置 token allowances：
- **USDC**: `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174`
- **Conditional Tokens**: `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045`

批准给这些合约:
- `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` (Main exchange)
- `0xC5d563A36AE78145C45a50134d48A1215220f80a` (Neg risk markets)
- `0xd91E80cF2E7be2e162c6513ceD06f1dD0dA35296` (Neg risk adapter)

Email/Magic 钱包用户无需手动设置。

---

## 📊 市场结构理解

### Polymarket 二元期权机制

每个预测问题有**两个 Token**：

| Token | 含义 | 结算价值 |
|-------|------|---------|
| **YES** | "这件事会发生" | 结果为 YES = $1.00，否则 $0.00 |
| **NO** | "这件事不会发生" | 结果为 NO = $1.00，否则 $0.00 |

**关键公式**：
```
YES 价格 + NO 价格 ≈ $1.00
```

---

## 🎯 核心策略框架

### 策略一：Smart Money 跟单（Copy Trading）

**核心原理**：跟随高胜率钱包，检测共识

**最佳实践（来自社区）**：

1. **钱包选择规则**
   - 排行榜 Top 50 高胜率地址
   - ROI > 30% 且历史交易 > 20 笔
   - 避免"一次性大奖"地址（靠单笔大赢）

2. **共识检测**
   - 至少 3-5 个 Smart Money 同时买入同方向
   - 时间窗口：1-5 分钟内
   - 金额门槛：总净流入 > $2000-5000

3. **信号过滤**
   ```
   ✅ 多人共识（3+ 人）+ 中等共识度（70-85%）
   ✅ 尾盘单人大额（1 人 + $500+ + 1 小时内到期）
   ❌ 单人小额（< $500）
   ❌ 过高共识（> 85%，可能已涨过头）
   ```

4. **Tiered Multipliers（分层乘数）**
   ```
   小额 (<$500):    0.3x → 减少噪音
   中额 ($500-$2000): 0.5x
   大额 ($2000-$10000): 0.8x
   超大额 (>$10000):  1.0x → 跟随大资金
   ```

5. **Copy Trading 模式选择（社区最佳实践 2026）**
   | 模式 | 适用场景 | 说明 |
   |------|---------|------|
   | **Fixed** | 日交易 < 5 笔 | 每笔固定金额 |
   | **Exact** | 资金规模相近 | 完全复制比例 |
   | **Portfolio** | 多笔小仓位 | 按组合比例 |

   **关键警告**: 
   - ❌ 不要跟做市商 (market makers)
   - ❌ 不要跟套利机器人 (arbitrage bots)
   - ✅ 标记钱包类型："AI trader", "sports 85% win rate whale"

---

### 策略二：NO Farming（做空过度炒作）

**核心原理**：大多数预测市场最终结果是 NO

**最佳实践**：

1. **入场条件**
   - YES 价格：$0.80-0.97
   - 过度炒作迹象（成交量异常高、讨论热度高）
   - 短期市场（< 7 天到期）

2. **风险控制**
   - 单笔限额：$5-10
   - 分散到 10+ 市场
   - 止损：-15%

3. **黑天鹅对冲**
   - 避免"奇迹可能发生"的市场
   - 政治/体育类更容易炒作（群众情绪化）

---

### 策略三：高概率复利（High-Probability Compounding）

**核心原理**：买入高概率合约，小额频繁交易

**最佳实践**：

1. **入场条件**
   - YES 价格：$0.90-0.99
   - 短期市场（< 24 小时）
   - 体育比赛、加密价格类

2. **执行**
   - 胜率 90%+，利润 1-10%
   - 高频重复，复利增长
   - 单笔金额：$5-20

---

### 策略四：Whale Tracking（大户追踪）

**核心原理**：追踪大额交易，识别 Smart Money 流向

**最佳实践**：

1. **数据来源**
   - Polymarket Data API (`/v1/leaderboard`, `/positions`, `/trades`)
   - 第三方工具：Polymarket Whale Tracker, Unusual Whales

2. **评分维度**
   ```
   Volume（交易量）: 权重 20%
   ROI（投资回报率）: 权重 40%
   PnL（盈亏）: 权重 40%
   ```

3. **过滤条件**
   - 至少 30% 高质量交易员支持
   - 盈利地址优先
   - 排除"一次性大奖"地址

---

### 策略五：Trailing Stop（追踪止损）

**核心原理**：买入低价方，追踪价格变动，买入另一方锁定利润

**来源**: [Rust-Polymarket-Trading-Bot](https://github.com/bitman09/Rust-Polymarket-Trading-Bot)

**最佳实践**：

1. **入场条件**
   - 只买入价格 < $0.50 的方（underdog）
   - 等待 trailing stop 触发（ask ≥ lowest + trailing_stop_point）
   - 支持 2-outcome 和 3-outcome 市场

2. **执行流程**
   ```
   第一步: 追踪所有 outcome，只买入价格 < 0.50 的方
   第二步: 追踪剩余 token，触发时买入
   循环: continuous=true 时重复直到市场结束
   ```

3. **配置参数**
   | 参数 | 推荐值 | 说明 |
   |------|--------|------|
   | trailing_stop_point | 0.03 | 价格反弹阈值 |
   | trailing_shares | 10 | 每次买入股数 |
   | check_interval_ms | 1000 | 检查间隔 |
   | sell_price | 0.99 | 卖出目标价 |

4. **风险控制**
   - 买入双方后锁定利润（无论结果）
   - 避免单边持仓风险
   - 适合低流动性市场

---

### 策略六：7种套利策略组合

**来源**: [Zeta-Trade/Polymarket-Trading-Bot](https://github.com/Zeta-Trade/Polymarket-Trading-Bot)

| 策略 | 原理 | 风险等级 |
|------|------|---------|
| **Liquidity Absorption Flip** | 吸收流动性后翻转价格 | 高 |
| **Orderbook Parity Arbitrage** | YES+NO<$1 时买入双方 | 低（已被费用侵蚀）|
| **Structural Spread Lock** | 利用恐慌错定价买入双方 | 低 |
| **Systematic NO Farming** | 系统性做空过度炒作 | 中 |
| **Long-Shot Floor Buying** | 买入最低价（≈0.1¢）的 YES | 高 |
| **Spread Farming** | 高频买卖赚取价差 | 低 |
| **High-Probability Auto-Compounding** | 买入 $0.90-0.99 合约复利 | 低 |

**关键警告 (poly-maker 作者)**:
> "In today's market, this bot is not profitable and will lose money. Given the increased competition on Polymarket, I don't see a point in playing with this unless you're willing to dedicate a significant amount of time."

**这意味着**:
- 简单策略已无效（竞争激烈）
- 需要 **adaptive execution**（自适应执行）
- 费用（3.15%）已侵蚀大部分套利利润

---

## 📉 价格区间分析

### 价格区间胜率（实战数据）

| 价格区间 | 胜率 | 策略建议 |
|----------|------|---------|
| $0.00-0.30 | 高风险 | Long-Shot Floor Buying |
| $0.30-0.50 | 中等 | ⚠️ 谨慎 |
| $0.50-0.70 | 最低 | ❌ 避免 |
| **$0.70-0.80** | **低** | **❌ 死亡区间** |
| $0.80-0.97 | 高 | ✅ NO Farming |
| $0.97-1.00 | 很高 | ✅ 高概率复利 |

**关键发现**：
- **$0.70-0.80 是死亡区间**（胜率仅 25%）
- **$0.80+ 是 NO Farming 黄金区**

---

## 🔧 风控原则

### Kelly Criterion 仓位管理

**公式**：
```
Kelly% = (Win% × Odds - Lose%) / Odds
Half-Kelly = Kelly% / 2
```

**推荐配置**：
- 使用 **Half-Kelly**（更保守）
- 单笔风险：**2-5%** bankroll
- 止损：**25-30%** drawdown
- 现金储备：**20%**

### 止损策略

| 入场价 | 止损幅度 |
|-------|---------|
| <$0.20 | -40% |
| $0.20-0.40 | -35% |
| $0.40-0.55 | -30% |
| $0.55-0.70 | -25% |
| $0.70-0.80 | -20% |
| $0.80-0.90 | -15% |
| $0.90-0.95 | -10% |
| ≥$0.95 | -7% |

### 持仓限制

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| 单笔金额 | $5-10 | 小额分散 |
| 最大持仓数 | 20 | 避免过度分散 |
| 单市场最大 | $50 | 限制敞口 |
| 总持仓上限 | $700 | 资金安全 |

---

## 🚫 黑名单市场类型

**避免交易**：
- BTC 价格类（"BTC 会涨到 $X"）— 难预测
- Spread/Over-Under — 庄家优势大
- "Up or Down" 类 — 纯赌博
- 极长期市场（> 30 天）— 不确定性高
- 死亡区间（$0.70-0.80）

---

## 🛠️ 技术实现最佳实践

### 1. 信号确认机制

```
非尾盘信号（价格 < $0.85）：
  - 5分钟内需要 2 次确认才执行
  
尾盘信号（价格 ≥ $0.85）：
  - 跳过确认，直接执行
```

### 2. 多窗口共识

| 信号强度 | 净流入 | 交易员数 | 确认窗口 |
|---------|--------|---------|---------|
| 大额即时 | ≥$5000 | - | 立即入场 |
| 大流入即时 | ≥$10000 | - | 立即入场 |
| 中等快速 | ≥$3000 | ≥5 | 60秒 |
| 标准 | ≥$200 | ≥2 | 300秒 |

### 3. Trade Aggregator（小额聚合）

```python
# 小额交易（< $1）进入 buffer 聚合
# 等待时间窗口后统一执行
# 避免手续费损耗
```

### 4. Slippage Protection（滑点保护）

```python
# 实时滑点估算
# 动态滑点阈值
# 流动性检查
# 价格冲击预测
```

---

## 📈 实战数据总结

### 当前策略表现

| 策略 | 执行笔数 | 盈亏 | 状态 |
|------|---------|------|------|
| Smart Money | 109 | -$403 | ❌ 已优化 |
| Copy Trading | 9 | +$12 | ✅ 保留 |
| NO Farming | 0 | $0 | 🔜 待启用 |

### 价格区间胜率

| 区间 | 胜率 | 交易数 |
|------|------|--------|
| High ($0.80-1.00) | **100%** | 8 |
| Mid ($0.60-0.80) | **0%** | 2 |

---

## 🆕 新增功能（2026-03-12）

### Whale Scorer（大户评分系统）

**功能**：评估交易员历史胜率，过滤低质量大户

**数据来源**：
- Polymarket Leaderboard API
- Positions API
- Trades API

**评分公式**：
```
score = 0.2 × (volume/1M) + 0.4 × ROI + 0.4 × (pnl/100K)
```

**配置**：
```bash
ENABLE_WHALE_SCORER=true
MIN_WHALE_RATIO=0.3  # 至少30%高质量交易员
```

### Consensus Checker（多信号共识）

**功能**：收集多个 Skill 信号，判断共识

**共识级别**：
| 级别 | 条件 |
|------|------|
| NONE | 无共识 |
| WEAK | 2个信号 |
| MODERATE | 3个信号 |
| STRONG | 4+信号 |
| UNANIMOUS | 全部同向 |

**配置**：
```bash
ENABLE_CONSENSUS_CHECK=true
```

---

## 📚 参考资料

### GitHub 开源项目

| 项目 | Stars | 核心价值 |
|------|-------|---------|
| [poly-maker](https://github.com/warproxxx/poly-maker) | 927 | Market Making |
| [Polymarket-Copy-Trading-Bot](https://github.com/ZeljkoMarinkovi/Polymarket-Copy-Trading-Bot) | 706 | Tiered Multipliers |
| [Polymarket-Trading-Bot](https://github.com/Zeta-Trade/Polymarket-Trading-Bot) | 344 | 7 种策略 |
| [polybot](https://github.com/ent0n29/polybot) | 199 | 反向工程策略 |
| [Rust-Polymarket-Trading-Bot](https://github.com/bitman09/Rust-Polymarket-Trading-Bot) | 184 | Trailing Stop |

### 社区资源

- [Polymarket Whale Tracker (Apify)](https://apify.com/jy-labs/polymarket-whale-tracker) - 大户追踪
- [PolyGun Bot](https://t.me/PolyGunSniperBot) - Telegram Copy Trading Bot
- [polymarket-strategy-advisor](https://lobehub.com/skills/mjunaidca-polymarket-skills-polymarket-strategy-advisor) - Half-Kelly 仓位管理
- [Oddpool](https://oddpool.com) - "Bloomberg of prediction markets" 多平台聚合
- [Whales Market](https://whales.market) - 大户监控工具

### 新增分析工具 (2026)

| 工具 | 功能 | 链接 |
|------|------|------|
| **Kreo** | 终端+分析 | 多种 Copy Trading 模式 |
| **Oddpool** | 多平台聚合 | 实时赔率、套利机会 |
| **Insider Finder** | 内幕检测 | 新钱包异常交易 |
| **UMA Control Panel** | 争议追踪 | 投票监控 |

### API 文档

- [Polymarket API Docs](https://docs.polymarket.com)
- [CLOB API Docs](https://docs.polymarket.com/developers/CLOB)
- [py-clob-client](https://github.com/Polymarket/py-clob-client)

---

## 💡 持续改进检查清单

- [ ] 定期回顾 Trade Journal（每日/每周）
- [ ] 分析胜率变化趋势
- [ ] 调整信号过滤参数
- [ ] 清理亏损持仓
- [ ] 保持策略纪律
- [ ] 更新黑名单关键词
- [ ] 监控大户排行榜变化
- [ ] 测试新策略（先用 Dry Run）

---

## 🔧 当前系统配置分析 (2026-03-14)

### 系统架构

```
run_automaton_integrated.py
    ├── AutomatonV2 (元策略管理器)
    │   ├── SmartMoneyServiceV2 (Smart Money 流)
    │   ├── Skills V2 策略
    │   │   ├── Mert Sniper
    │   │   ├── Signal Sniper
    │   │   ├── AI Divergence
    │   │   ├── CopyTrading
    │   │   └── Weather Trader
    │   └── Bandit Optimizer (动态权重)
    ├── Position Monitor (持仓监控)
    └── Param Manager (参数管理)
```

### 当前配置参数

| 参数 | 当前值 | 说明 |
|------|--------|------|
| `ENABLED_DETECTORS` | `["smart_closing"]` | 仅启用 smart_closing |
| `AUTO_TRADE_ENABLED` | `false` | 自动交易关闭 |
| `SIGNAL_TRADE_AMOUNT_MIN` | `$1.0` | 最小交易金额 |
| `SIGNAL_TRADE_AMOUNT_MAX` | `$5.0` | 最大交易金额 |
| `MIN_CONFIDENCE` | `0.60` | 最小信号置信度 60% |
| `MAX_POSITION_PER_MARKET` | `$15` | 单市场持仓限制 |
| `SIGNAL_CONFIRMATION_COUNT` | `3` | 需要 3 次确认 |
| `MIN_TIME_TO_EXPIRY` | `6h` | 最小到期时间 |

### Bandit 算法配置

```python
# automaton_v2.py
exploration_weight = 2.0      # 探索权重
min_pulls_before_exploit = 3  # 最小执行次数后才利用
```

---

## 💡 优化建议 (2026-03-14)

### 1. 🎯 策略层优化

#### 当前问题
- 仅启用 `smart_closing` 一个检测器，策略多样性不足
- `SIGNAL_CONFIRMATION_COUNT=3` 门槛过高，可能错过快速信号

#### 建议
```bash
# 启用更多检测器
ENABLED_DETECTORS=["smart_closing", "whale", "orderbook"]

# 降低确认门槛，提高灵敏度
SIGNAL_CONFIRMATION_COUNT=2
SIGNAL_CONFIRMATION_WINDOW=180  # 缩短确认窗口
```

### 2. 💰 交易金额优化

#### 当前问题
- `$1-5` 金额过小，手续费侵蚀利润
- 3.15% 手续费下，$5 交易成本 $0.16

#### 建议
```bash
# 提高交易金额
SIGNAL_TRADE_AMOUNT_MIN=3.0
SIGNAL_TRADE_AMOUNT_MAX=10.0

# 高置信度信号加仓
# 在代码中实现: confidence > 0.8 时 amount *= 1.5
```

### 3. 📊 价格区间优化

#### 当前问题
- `SIGNAL_TRADE_MIN_PRICE=0.50` 允许中价位入场
- 根据实战数据，$0.70-0.80 是死亡区间

#### 建议
```bash
# 避开死亡区间
SIGNAL_TRADE_MIN_PRICE=0.80  # 只做高价区（NO Farming）
# 或者
SIGNAL_TRADE_MAX_PRICE=0.70  # 只做低价区（Long-Shot）
```

### 4. 🔥 高优先级改进

| 优先级 | 改进项 | 预期效果 |
|--------|--------|---------|
| P0 | 启用 `whale` 检测器 | 捕获大额交易信号 |
| P0 | 降低 `SIGNAL_CONFIRMATION_COUNT` 到 2 | 提高信号执行率 |
| P1 | 实现 Tiered Multipliers | 大额信号加仓 |
| P1 | 添加死亡区间过滤 | 避开 $0.70-0.80 |
| P2 | 实现 Trade Aggregator | 小额聚合降低手续费 |
| P2 | 添加 Slippage Protection | 动态滑点控制 |

### 5. 🧪 测试建议

1. **先 Dry Run 测试**: 设置 `AUTO_TRADE_DRY_RUN=true`
2. **小资金试运行**: `amount_min=1, amount_max=3`
3. **观察 Bandit 权重变化**: 监控各策略 UCB 分数
4. **逐步启用策略**: 先 smart_closing，稳定后加 whale

---

## 📈 实战数据监控

### 建议监控指标

```python
# 每日统计
- 总信号数
- 执行信号数
- 胜率
- 平均盈亏
- Bandit 权重分布
- 策略 P&L 排名
```

### 告警阈值

| 指标 | 阈值 | 动作 |
|------|------|------|
| 连续亏损 | > 5 笔 | 暂停交易，检查策略 |
| 单日亏损 | > $50 | 暂停交易，人工审查 |
| 胜率 | < 40% | 调整参数或禁用策略 |
| 持仓超限 | > $700 | 停止开新仓 |

---

*本文档基于开源项目研究、社区最佳实践和实战数据持续更新*