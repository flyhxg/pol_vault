# Polymarket 交易策略最佳实践

> 基于 GitHub 开源项目研究、实战经验总结和社区最佳实践
> 更新时间: 2026-03-18 12:33 UTC

---

## 🆕 最新更新 (2026-03-18 12:33)

### ✅ 关键配置问题已修复

**问题**: 尾盘信号 (Stage5/6, 价格 0.50-0.70) 全部被价格过滤器拦截

**根因**:
1. `.env`: `SIGNAL_TRADE_MIN_PRICE=0.80` 阻止低价信号
2. `run_automaton_integrated.py` 第 173 行硬编码 `if price >= 0.80`

**已修复**:
```bash
# 1. .env 配置
SIGNAL_TRADE_MIN_PRICE=0.50  # 从 0.80 改为 0.50

# 2. run_automaton_integrated.py 代码
# 移除硬编码 price >= 0.80，改为读取 MIN_CONFIDENCE 环境变量
min_confidence = float(os.getenv('MIN_CONFIDENCE', '0.6'))
if analysis.get('confidence', 0) >= min_confidence:
    # 允许所有价格信号通过
```

### 当前生效配置 (2026-03-18 12:33)

**价格范围**:
| 参数 | 值 | 说明 |
|------|-----|------|
| `SIGNAL_TRADE_MIN_PRICE` | **0.90** | 只保留 Stage1/2 |
| `SIGNAL_TRADE_MAX_PRICE` | 0.99 | 最高价格 |
| 死亡区间 | 0.70-0.80 | ❌ 硬编码过滤 |

**Smart Money Stage 净流入门槛** (只保留 Stage1/2):
| Stage | 价格区间 | 净流入门槛 |
|-------|---------|-----------|
| Stage1 | 0.95-0.999 | $1,000 |
| Stage2 | 0.90-0.95 | $500 |

**策略分工** (按价格区间):
| 价格区间 | 策略 | 触发条件 |
|---------|------|---------|
| < 0.50 | Trailing Stop | Underdog 反弹 |
| 0.80-0.90 | Tail Strategy | 趋势 + 成交量飙升 |
| ≥ 0.90 | Smart Money | Stage1/2 净流入 |

**过滤参数**:
| 参数 | 值 |
|------|-----|
| `WEBSOCKET_MICRO_TRADE_THRESHOLD` | $50 |
| `HARD_LIMIT_MIN_NET_FLOW` | $100 |
| `MIN_CONFIDENCE` | 60% |
| `MIN_TIME_TO_EXPIRY` | 6h |

**启用的策略**:
- ✅ copytrading (Smart Money 跟单)
- ✅ tail_strategy (趋势追踪 ≥ 0.80)
- ✅ trailing_stop (Underdog 抄底)
- ✅ trade_journal (交易日志)

### 7 种套利策略详解 (Zeta-Trade)

| 策略 | 核心逻辑 | 当前费用环境下 | 推荐度 |
|------|---------|--------------|--------|
| **Liquidity Absorption Flip** | 吸收流动性后翻转价格 | 可行 | ⭐⭐⭐ |
| **Orderbook Parity Arbitrage** | YES+NO<$1 买入双方 | ❌ 已被费用侵蚀 | ❌ |
| **Structural Spread Lock** | 恐慌错定价买入双方 | 可行 | ⭐⭐⭐ |
| **Systematic NO Farming** | 系统性做空过度炒作 | ✅ 有效 | ⭐⭐⭐⭐⭐ |
| **Long-Shot Floor Buying** | 买最低价 YES (≈0.1¢) | 高风险高回报 | ⭐⭐ |
| **Spread Farming** | 高频买卖赚价差 | ❌ 费用侵蚀 | ❌ |
| **High-Probability Compounding** | 买 $0.90-0.99 复利 | ✅ 有效 | ⭐⭐⭐⭐ |

**关键警告**: 3.15% 手续费已侵蚀大部分简单套利利润，需要 **Adaptive Execution**（自适应执行）

### Trailing Stop 策略最佳实践 (bitman09/Rust-Polymarket-Trading-Bot)

**核心原理**: 买入 Underdog (<$0.50)，追踪价格反弹，买入另一方锁定利润

**配置参数**:
| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `trailing_stop_point` | 0.03 | 价格反弹阈值 |
| `trailing_shares` | 10 | 每次买入股数 |
| `check_interval_ms` | 1000 | 检查间隔 |
| `continuous` | true | 重复直到市场结束 |
| `sell_price` | 0.99 | 卖出目标价 |

**执行流程**:
```
第一步: 追踪所有 outcome，只买价格 < 0.50 的方 (underdog)
第二步: 追踪剩余 token，触发时买入
循环: continuous=true 时重复直到市场结束
```

**支持**: 2-outcome 和 3-outcome 市场

### 当前系统状态

| 指标 | 数值 |
|------|------|
| 进程运行时间 | ~7 小时 |
| 总信号接收 | 6 |
| 信号执行 | 0 (全部被价格过滤) |
| 持仓数量 | 19 个 |
| 总投入 | ~$90 |

---

## 🆕 最新更新 (2026-03-17 12:00)

### Polymarket 官方 API 最佳实践

**核心 SDK**: `@polymarket/clob-client` / `py-clob-client`

**推荐架构**:
```python
from py_clob_client.client import ClobClient

HOST = "https://clob.polymarket.com"
CHAIN_ID = 137  # Polygon Mainnet

# 市价单执行（推荐 FOK - Fill or Kill）
order = MarketOrderArgs(
    token_id=token_id, 
    amount=25.0, 
    side=BUY, 
    order_type=OrderType.FOK
)
```

### 策略代码审核完成 (2026-03-17 05:35)

**已删除的禁用策略**:
- mert_sniper_v2.py (尾盘狙击)
- ai_divergence_v2.py (AI 分歧检测)
- signal_sniper_v2.py (RSS 新闻信号)
- weather_trader_v2.py (天气事件交易)
- fast_loop_v2.py (BTC/ETH/SOL 市场)

**保留的核心策略**:
| 策略 | 核心逻辑 | 状态 |
|------|---------|------|
| **Smart Money** | 跟踪大额交易流，检测共识 | ✅ 运行中 |
| **Copytrading** | 查询历史交易，共识/单人大额 | ✅ 运行中 |
| **Tail Strategy** | 追踪强势方 (价格 > 80%) | ✅ 已修复 bug |
| **Trailing Stop** | Underdog 抄底，等待价格反弹 | ✅ 运行中 |

### Tail Strategy Bug 修复 (2026-03-17 05:35)

**问题**:
- 配置追踪 >80% 的市场，但买入 ~35% 的 YES
- 价格获取依赖 WebSocket 缓存，经常为空

**修复**:
1. 添加 REST API fallback（和 trailing_stop 一致）
2. 修正核心逻辑：
   - YES 价格 > 80% → 买 YES
   - NO 价格 > 80% → 买 NO
3. 传入 SDK 参数用于 API 调用

### 当前进程状态

**运行时间**: ~4 小时（08:01 UTC 重启）
**PID**: 4103912
**内存**: ~710 MB

---

## 🆕 最新更新 (2026-03-17 02:22)

### 当前持仓快照

**运行状态**: 进程刚重启（02:20 UTC）

| 指标 | 数值 |
|------|------|
| 持仓数量 | **17 个** |
| 总成本 | $134.17 |
| 总价值 | $141.37 |
| **浮动盈亏** | **+$7.20 (+5.4%)** |

**Top 盈利持仓**:
| 市场 | 入场价 | 当前价 | 盈亏 |
|------|--------|--------|------|
| US forces enter Iran | $0.573 | $0.75 | **+$2.96** |
| Ken Paxton primary | $0.46 | $0.61 | **+$1.56** |
| Russia-Ukraine ceasefire | $0.862 | $0.981 | **+$1.10** |
| NBA DAL-MIL | $0.53 | $0.63 | **+$0.99** |

**亏损持仓**:
| 市场 | 入场价 | 当前价 | 盈亏 |
|------|--------|--------|------|
| Russia invade NATO | $0.957 | $0.947 | -$0.31 |
| MegaETH airdrop | $0.658 | $0.63 | -$0.20 |

### 实际启用的策略

| 策略 | 状态 | 功能 |
|------|------|------|
| `copytrading` | ✅ 运行中 | Smart Money 跟单（核心） |
| `tail_strategy` | ✅ 运行中 | 趋势追踪 |
| `trailing_stop` | ✅ 运行中 | 追踪止损 |
| `trade_journal` | ✅ 运行中 | 交易日志 |

### 已禁用的策略

| 策略 | 功能 |
|------|------|
| `mert_sniper` | 尾盘狙击 |
| `fast_loop` | BTC/ETH/SOL 市场 |
| `signal_sniper` | RSS 新闻信号 |
| `ai_divergence` | AI 分歧度检测 |
| `weather_trader` | 天气事件交易 |

### 已删除的策略

- `no_farming.py` — Polymarket 没有"做空"概念，只有 YES/NO Token

### 待调整配置

**问题**: `MIN_TIME_TO_EXPIRY=6` 过滤掉大部分热门市场

**日志证据**:
```
[FILTERED] ❌ Too Early: 333.6h > 6h | Market: Trump out as President...
[FILTERED] ❌ Too Early: 6933.6h > 6h | Market: Netanyahu out by March 31...
```

**建议**: 改成 `MIN_TIME_TO_EXPIRY=24`

**状态**: ❌ 未调整

---

## 🆕 最新更新 (2026-03-16 12:00)

### py-clob-client 官方 SDK 更新

**核心要点**：
- 签名类型区分：
  - `signature_type=0`: MetaMask/硬件钱包（需设置 Token Allowances）
  - `signature_type=1`: Email/Magic 钱包
  - `signature_type=2`: Browser wallet proxy

**关键 API 端点**：
```python
HOST = "https://clob.polymarket.com"
CHAIN_ID = 137  # Polygon Mainnet

# 获取市场数据
midpoint = client.get_midpoint(token_id)
price = client.get_price(token_id, side="BUY")
book = client.get_order_book(token_id)
```

### poly-maker 作者警告（重要）

> ⚠️ **"In today's market, this bot is not profitable and will lose money."**

**这意味着**：
- 简单做市策略已失效（竞争激烈）
- 3.15% 手续费侵蚀大部分利润
- 需要 **adaptive execution**（自适应执行）
- 应该专注于 **Smart Money 跟单** 和 **NO Farming**

### 当前系统配置快照

| 参数 | 当前值 | 建议值 | 说明 |
|------|--------|--------|------|
| `SIGNAL_TRADE_MIN_PRICE` | **0.80** ✅ | 0.80 | 高价区已设置 |
| `SIGNAL_TRADE_MAX_PRICE` | 0.99 | 0.99 | 正确 |
| `SIGNAL_TRADE_AMOUNT_MIN` | $1.0 | **$3.0** | 手续费侵蚀 |
| `SIGNAL_TRADE_AMOUNT_MAX` | $5.0 | **$10.0** | 提高资金效率 |
| `SIGNAL_CONFIRMATION_COUNT` | 3 | **2** | 提高执行率 |
| `COOLDOWN_AFTER_LOSS_STREAK` | **3** ✅ | 3 | 已启用 |
| `MIN_WHALE_RATIO` | 0.30 | 0.30 | 正确 |
| `AUTO_TRADE_ENABLED` | **false** | true | 需手动开启 |

### 实战数据洞察（最近 24 小时）

**价格区间胜率**：
| 区间 | 胜率 | 交易数 | 结论 |
|------|------|--------|------|
| $0.80-1.00 | **96.0%** | 24/25 | ✅ 黄金区间 |
| $0.00-0.60 | **0.0%** | 0/19 | ❌ 避免 |
| $0.60-0.80 | **0.0%** | 0/2 | ❌ 死亡区间 |

**类别胜率**：
| 类别 | 胜率 | 战绩 |
|------|------|------|
| Sports | **63.2%** | 12/19 |
| Entertainment | 66.7% | 2/3 |
| Other | 43.5% | 10/23 |
| Politics | 0.0% | 0/1 |

### 进程运行状态

- 运行时间：**~120 小时**（Mar15 启动）
- 进程稳定，无崩溃
- 当前状态：W3（连胜3）

---

## 🆕 最新更新 (2026-03-15 12:00)

### 代码审查关键发现

**1. Sweeping（尾盘）阶段限制已解除**

之前代码中有禁止 Sweeping 阶段交易的逻辑：
```python
# 旧代码（已删除）
if stage == 'sweeping':
    logger.warning("Sweeping stage: Telegram only, NO auto-trade")
```

现已修改为允许所有阶段交易。Sweeping 配置：
- 时间：≤ 1 小时到期
- 价格：$0.95 - $0.999
- 单窗口触发（无需多窗口确认）

**2. 过滤逻辑架构清晰化**

```
交易事件 → Safeguards(市场质量) → Micro-trade($50) → 时间检查(≤6h) 
        → 价格分层 → 交易员数量 → Whale质量(≥30%) → 最终信号
```

**3. 配置冗余问题**

发现 `CopyTradingConfig.dry_run` 字段未被使用，实际下单由 `Automaton V2` 的 `AUTO_TRADE_DRY_RUN` 控制。

### 进程稳定性

- 当前进程稳定运行在 screen 会话 `55526.pol`
- 已运行 2+ 小时无崩溃
- 之前频繁重启是终端操作导致，非代码 bug

---

## 🆕 最新更新 (2026-03-15 00:10)

### 实战数据复盘

**SmartMoney 策略运行统计**:
```
执行信号: 133
胜/负: 13,845 / 17,466
胜率: 44.2%
净盈亏: +$767.59
```

**关键发现**:
1. **高价区 ($0.80-1.00) 胜率 100%** — 坚持这个区间
2. **低价区 ($0.00-0.60) 胜率 0%** — 避免入场
3. **死亡区间 ($0.70-0.80) 胜率极低** — 需要过滤

### 进程稳定性问题

**现象**: 进程约每 30 分钟崩溃一次
**原因**: 疑似内存泄漏
**临时方案**: 自动重启机制
**长期方案**: 排查内存泄漏源码

### 配置优化记录

已调整参数:
```bash
QUALITY_SCORE_MIN_TOTAL=50           # 质量分阈值 60→50
SAFEGUARDS_MIN_LIQUIDITY=5000        # 流动性门槛 10000→5000
WEBSOCKET_MICRO_TRADE_THRESHOLD=50   # Micro-trade 门槛 200→50
MIN_WHALE_RATIO=0.15                 # Whale 比例 30%→15%
SMART_MONEY_STAGE_1_MIN_NET_FLOW=1000  # Belief Locked 净流入 2000→1000
```

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

| 策略           | 执行笔数 | 盈亏    | 状态     |
| ------------ | ---- | ----- | ------ |
| Smart Money  | 109  | -$403 | ❌ 已优化  |
| Copy Trading | 9    | +$12  | ✅ 保留   |
| NO Farming   | 0    | $0    | 🔜 待启用 |

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

## 💡 优化建议 (2026-03-15)

### 🎯 基于实战数据的建议

#### 1. 价格区间策略调整

**当前问题**:
- 低价区 ($0.00-0.60) 0% 胜率
- 死亡区间 ($0.70-0.80) 极低胜率

**建议**:
```bash
# 只做高价区 NO Farming
SIGNAL_TRADE_MIN_PRICE=0.80
SIGNAL_TRADE_MAX_PRICE=0.99
```

#### 2. 进程稳定性优化

**问题**: 进程每 ~30 分钟崩溃一次

**排查方向**:
```python
# 检查内存泄漏
# 1. trade buffer 无限增长
# 2. market cache 未清理
# 3. 事件监听器重复注册
```

**临时方案**: 使用 systemd 或 supervisor 自动重启

#### 3. 胜率提升策略

**当前胜率 44.2%**，需要提升到 50%+

**建议**:
1. 只在高价区 ($0.80+) 交易
2. 增加 Whale 共识门槛到 3 人
3. 启用 `SIGNAL_CONFIRMATION_COUNT=2`

#### 4. 资金利用率

**当前持仓**: 3 个市场，总价值 ~$65

**建议**:
- 提高单笔金额到 $5-10
- 最大持仓数提高到 10 个
- 分散到不同类别市场

### 🔥 高优先级改进

| 优先级 | 改进项 | 预期效果 |
|--------|--------|---------|
| P0 | 价格区间限制到 $0.80+ | 胜率提升到 60%+ |
| P0 | 排查内存泄漏 | 进程稳定运行 |
| P1 | 提高单笔金额到 $5-10 | 提高资金利用率 |
| P1 | 启用 whale 检测器 | 捕获大额信号 |
| P2 | 实现 Trailing Stop | 锁定利润 |

---

## 🔧 当前代码配置分析 (2026-03-15 12:00)

### 实际生效配置

基于代码审查，当前生效的配置：

| 参数 | 配置值 | 来源 |
|------|--------|------|
| `AUTO_TRADE_DRY_RUN` | `false` | .env |
| `POSITION_MONITOR_DRY_RUN` | `false` | .env |
| `MIN_TIME_TO_EXPIRY` | `6h` | CopyTradingConfig |
| `SAFEGUARDS_MIN_LIQUIDITY` | `$5000` | 环境变量 |

### Copy Trading 配置（run_automaton_integrated.py）

```python
top_n = 50              # 跟踪前50个高胜率钱包
min_trade_size = $5     # 最小交易金额
max_size_per_trade = $10
window_size = 30秒      # 快速响应窗口
min_net_flow = $1000    # 最小净流入
min_trader_quality = 50 # 最低交易员质量分
min_trader_count = 2    # 最少参与交易员
min_price = $0.80       # 只跟高价区
max_price = $0.99
```

### Safeguards 配置

```python
min_liquidity = $5000   # 最小流动性
max_spread = 3%         # 最大价差
min_price = $0.50       # 最低价格
max_price = $0.99       # 最高价格
```

### 三阶段策略配置

| 阶段 | 时间限制 | 价格区间 | 最小交易员 |
|------|---------|---------|-----------|
| Early | ≤8h | $0.30-0.70 | 3 |
| Forming | ≤3h | $0.75-0.90 | 5 |
| Sweeping | ≤1h | $0.95-0.999 | 1 |

### 架构问题

**发现的问题**：

1. **冗余配置**：
   - Smart Money 配置了超高门槛（实际被禁用）
   - `CopyTradingConfig.dry_run` 字段未被使用

2. **双重服务**：
   - 两个 `SmartMoneyServiceV2` 实例同时运行
   - 都监听同样的 WebSocket 流

3. **文档不一致**：
   - 文档说 Sweeping "1-12小时"
   - 实际配置是 ≤1 小时

### 优化建议

**立即执行**：

1. 删除 Smart Money 的冗余配置
2. 统一 dry_run 配置逻辑
3. 更新文档与代码一致

**中期改进**：

1. 合并两个 SmartMoneyServiceV2 实例
2. 实现持仓时间限制（目前只是建议）
3. 添加 Sweeping 阶段的止损策略

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

## 🆕 最新更新 (2026-03-16 00:00)

### 实战数据分析（Trade Journal）

**24小时交易统计**:
```
交易数量: 78笔
胜率: 47.2%
总盈亏: $+0.00
```

**价格区间胜率（关键发现）**:
| 价格区间 | 胜率 | 战绩 | 结论 |
|----------|------|------|------|
| $0.00-0.60 | **0.0%** | 0/16 | ❌ **全败** |
| $0.60-0.80 | **0.0%** | 0/2 | ❌ 避免 |
| $0.80-1.00 | **94.4%** | 17/18 | ✅ **黄金区间** |

**优化措施已执行**:
```bash
SIGNAL_TRADE_MIN_PRICE=0.80  # 从 0.50 提升到 0.80
COOLDOWN_AFTER_LOSS_STREAK=3  # 连败3次后暂停
```

### 当前持仓分析（23个持仓）

**总投入**: ~$200
**当前价值**: ~$220
**浮动盈亏**: **+$20** (+10%)

**最佳持仓**:
| 市场 | 入场价 | 当前价 | 盈亏 |
|------|--------|--------|------|
| Russia-Ukraine ceasefire | 0.86 → 0.98 | +$1.1 |
| US forces enter Iran | 0.57 → 0.71 | +$2.3 |
| BRA-CRU 足球 | 0.51 → 0.76 | +$2.3 |

**亏损持仓**:
| 市场 | 入场价 | 当前价 | 盈亏 |
|------|--------|--------|------|
| Russia invade NATO | 0.96 → 0.95 | -$0.31 |
| MegaETH airdrop | 0.66 → 0.61 | -$0.34 |

### Smart Money vs Copy Trading 价格限制

| 模块 | 价格范围 | 来源 |
|------|----------|------|
| **Smart Money** | 0.80-0.99 | `SIGNAL_TRADE_MIN_PRICE=0.80` ✅ |
| **Copy Trading** | 0.50-0.99 + 跳过 0.70-0.80 | 硬编码 |

**发现**: Copy Trading 使用硬编码的价格过滤（第 944-950 行），`COPY_TRADING_MIN_PRICE` 环境变量未被实际使用。

### 架构发现

**过滤流程（9步）**:
```
WebSocket → 金额过滤($200) → 爆发拆单过滤 → 窗口统计(60s) 
         → 硬性门槛($100净流入) → 多窗口确认(3个) 
         → 阶段检测(价格区间) → 综合评分(50-65分) 
         → Safeguards($5k流动性) → 生成信号
```

**阶段配置（基于价格，非时间）**:
| 阶段 | 价格区间 | 时间限制 | 净流入门槛 | 最小交易员 | 最小评分 |
|------|----------|---------|-----------|----------|---------|
| Early | 0.30-0.70 | ≤8h | $150 | 3 | 50 |
| Forming | 0.70-0.95 | ≤3h | $600 | 5 | 60 |
| Sweeping | 0.95-0.999 | ≤1h | $300 | 1 | 65 |

---

## 💡 最新优化建议 (2026-03-16)

### 🎯 高优先级

| 优先级 | 建议 | 预期效果 |
|--------|------|---------|
| **P0** | 已执行: `SIGNAL_TRADE_MIN_PRICE=0.80` | 胜率从 47% → 预计 60%+ |
| **P0** | 已执行: `COOLDOWN_AFTER_LOSS_STREAK=3` | 减少连败亏损 |
| **P1** | 提高 `SIGNAL_TRADE_AMOUNT_MIN=3.0` | 减少手续费侵蚀 |
| **P1** | 统一 Copy Trading 价格限制到 0.80 | 与 Smart Money 一致 |
| **P2** | 实现 Trailing Stop | 锁定利润 |

### 📊 数据驱动的结论

1. **高价区间 ($0.80-1.00) 胜率 94.4%** — 这是 NO Farming 黄金区
2. **低价区间 ($0.00-0.60) 胜率 0%** — 应完全避免
3. **死亡区间 ($0.70-0.80)** — Copy Trading 已硬编码跳过
4. **当前胜率 47.2%** — 需提升到 50%+，提高价格门槛是关键

### 🔧 待优化配置

```bash
# 建议调整
SIGNAL_TRADE_AMOUNT_MIN=3.0      # 从 $1 提升到 $3
SIGNAL_TRADE_AMOUNT_MAX=10.0     # 从 $5 提升到 $10

# Copy Trading 需要修改代码（第 944 行）
# 从 0.50 改为 0.80
```

---

*本文档基于开源项目研究、社区最佳实践和实战数据持续更新*