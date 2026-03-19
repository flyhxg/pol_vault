# Polymarket 交易策略最佳实践

## 1. 价格区间策略

### 1.1 高价格区间 (≥0.90) - 最佳表现

**胜率数据**:
| 价格区间 | 胜率 | EV | 建议 |
|---------|------|-----|------|
| 0.95-1.00 | 89.5% | +$0.95 | ✅ 优先交易 |
| 0.85-0.95 | 75% | +$0.60 | ✅ 可交易 |
| 0.70-0.80 | 25% | -$0.39 | ❌ 死亡区间 |
| <0.50 | 35% | -$0.22 | ⚠️ 仅限抄底 |

**最佳实践**:
- **尾盘狙击**: 价格 ≥ 0.95，剩余时间 < 15分钟 → 立即买入
- **置信度阈值**: 60% 以上才执行
- **窗口确认**: 尾盘只需 1 个窗口 (30秒)，其他需要 3 个窗口

### 1.2 死亡区间 (0.70-0.80)

**必须避开**:
- 胜率仅 25%
- EV 为负 (-$0.39)
- 所有策略应自动过滤此区间

---

## 2. 订单流分析策略

### 2.1 Smart Money 跟踪

**核心指标**:
| 指标 | 阈值 | 说明 |
|------|------|------|
| 单笔大单 | ≥$5,000 | WHALE_SNIPER 信号 |
| 净流入 | ≥$1,500 (尾盘) | 强势信号 |
| 共识人数 | ≥3人 | CONSENSUS_FLOW |
| 交易员质量 | >30% 高质量 | Whale Scorer |

**窗口确认机制**:
```
尾盘 (≥0.95):        1 个窗口 = 30秒
高置信度 (0.85-0.95): 2 个窗口 = 60秒  
其他:                3 个窗口 = 90秒

豁免条件:
- 单笔 ≥$5,000 → 立即触发
- 净流入 ≥$10,000 → 立即触发
- 中等信号 ($3,000+ + 5人) → 2 窗口
```

### 2.2 信号质量评分

**评分维度** (0-100):
- 方向一致性: 30%
- 净流入金额: 25%
- 交易员数量: 20%
- 时间一致性: 15%
- 行为分析: 10%

**通过阈值**: ≥50 分

---

## 3. 风险控制

### 3.1 交易限制

```bash
# 价格范围
SIGNAL_TRADE_MIN_PRICE=0.90
SIGNAL_TRADE_MAX_PRICE=0.99

# 置信度
MIN_CONFIDENCE=0.60

# 价差
MAX_SPREAD=0.03  # 3%

# 流动性
SAFEGUARDS_MIN_LIQUIDITY=10000  # $10,000

# 冷却时间
COOLDOWN_SECONDS=3600  # 1小时
```

### 3.2 关键词过滤

**屏蔽的市场类型**:
- Up or Down
- Spread
- O/U (Over/Under)
- Total

### 3.3 滑点保护

```bash
MAX_SLIPPAGE_PCT=3.0    # 最大滑点 3%
WARNING_SLIPPAGE_PCT=1.5  # 警告阈值
MIN_LIQUIDITY_USD=50.0   # 最小流动性
```

---

## 4. 市场选择

### 4.1 类别优先级

| 类别 | 胜率 | 优先级 |
|------|------|--------|
| Sports | 80% | ⭐⭐⭐ 优先 |
| Crypto | 67% | ⭐⭐ 可选 |
| Other | 43% | ⭐ 慎重 |

### 4.2 流动性要求

- 最小流动性: $10,000
- 最小 24h 成交量: $5,000
- 最大价差: 3%

---

## 5. 执行优化

### 5.1 订单类型

| 场景 | 订单类型 | 说明 |
|------|---------|------|
| 高确定性 | FOK (Fill or Kill) | 立即成交或取消 |
| 流动性差 | GTC (Good Till Cancel) | 挂单等待 |
| 大额交易 | 拆单执行 | 减少滑点 |

### 5.2 金额分配

```bash
SIGNAL_TRADE_AMOUNT_MIN=1.0  # 最小 $1
SIGNAL_TRADE_AMOUNT_MAX=5.0  # 最大 $5

# 动态调整:
# - 尾盘信号: 金额 × 1.3
# - 大单信号: 金额 × 1.5
```

---

## 6. 数据源

### 6.1 实时数据

- **WebSocket**: Polymarket CLOB 实时订单流
- **价格更新**: 30秒窗口聚合
- **市场缓存**: 6小时刷新周期

### 6.2 API 端点

```bash
# CLOB API
CLOB_API_URL=https://clob.polymarket.com

# Gamma API (市场信息)
GAMMA_API_URL=https://gamma-api.polymarket.com

# Data API
DATA_API_URL=https://data-api.polymarket.com
```

---

## 7. 监控与告警

### 7.1 进程监控

```bash
# 每 5 分钟检查进程
ps aux | grep run_automaton_integrated
```

### 7.2 关键日志

```bash
# 信号日志
[WINDOW-CONFIRM] Window X/Y
[SmartMoney] Emitting outcome_analysis event

# 错误日志
[FILTERED] 价格超出范围
[FILTERED] 净流入不足
```

---

## 8. 已知问题与解决方案

### 8.1 信号数量少

**原因**:
- 净流入门槛太高 ($1,500)
- 多窗口确认太严格

**解决**:
- 尾盘降低至 $500 (可选)
- 尾盘只需 1 个窗口确认

### 8.2 死亡区间信号

**原因**:
- Automaton V2 会自动过滤 0.70-0.80

**解决**:
- 已在 automaton_v2.py 内置过滤
- 置信度不足的信号自动跳过

---

## 9. 策略组合建议

### 9.1 推荐配置

```
尾盘 (≥0.95):
  ├─ WHALE_SNIPER: 大单立即触发
  ├─ SWEEP_DETECTED: 1 窗口确认
  └─ 金额: $3-5

高置信度 (0.85-0.95):
  ├─ CONSENSUS_FLOW: 2 窗口确认
  └─ 金额: $1-3

其他区间:
  ├─ 需 3 窗口确认
  └─ 金额: $1-2
```

### 9.2 避免的配置

- ❌ 在 0.70-0.80 价格区间交易
- ❌ 价差 > 3% 的市场
- ❌ 流动性 < $10,000 的市场
- ❌ 置信度 < 60% 的信号

---

## 10. 参考资料

- Polymarket Docs: https://docs.polymarket.com
- CLOB Client: https://github.com/Polymarket/clob-client
- Valory Trader: https://github.com/valory-xyz/trader
- Endcycle Sniper: https://github.com/Gabagool2-2/polymarket-trading-bot-python

---

**最后更新**: 2026-03-19
**版本**: 1.0