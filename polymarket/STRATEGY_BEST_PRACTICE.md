# Polymarket 策略最佳实践

> 最后更新: 2026-03-11
> 来源: Tavily 搜索 + 项目实战经验

---

## 核心策略

### 1. 资金管理
- **单笔仓位**: 75% 资金集中在高胜率交易 (75%+ odds)
- **到期时间**: 选择 1-2 天内到期的市场（时间价值衰减快）
- **风险分散**: 不要把所有资金分散在太多小仓位

### 2. 主要交易策略

| 策略 | 核心优势 | 风险 | 资金要求 | 执行速度 |
|------|---------|------|---------|---------|
| **套利 (Arbitrage)** | 无风险利润 | 窗口期短 | 中等 | 极快 |
| **鲸鱼跟单** | 利用大户信息优势 | 滑点大 | 低 | 中等 |
| **做市 (Market Making)** | 赚取价差 | 库存风险 | 高 | 持续 |
| **信息套利** | 信息优势 | 信息过时 | 低 | 快 |
| **Bot 自动化** | 24/7 执行 | 技术风险 | 中等 | 自动 |

### 3. 风险管理

#### 价格区间过滤
- **死亡区间**: [0.70-0.80] 胜率仅 25%，避免进入
- **最佳区间**: [0.80-0.95] 胜率高，期望值正
- **最低门槛**: 0.50（低于此价格不确定性太高）

#### 价差控制
- **最大价差**: 3%（超过则跳过）
- **滑点控制**: 1% 以内
- **流动性检查**: 确保有足够订单簿深度

#### 仓位管理
- **单笔最小**: $1（Polymarket 限制）
- **单笔最大**: $5-10（根据策略）
- **总持仓限制**: $700
- **单市场限制**: $30-50

### 4. 市场选择

#### 推荐类别
- **政治**: 信息透明，流动性好
- **加密**: 熟悉度高，波动大
- **体育**: 结果确定性强

#### 避免类别
- **O/U (大小分)**: 难预测
- **Spread (点差)**: 复杂度高
- **短期涨跌 (up/down)**: 噪音大

#### 关键词过滤
```
BLOCKED_KEYWORDS=["O/U","Spread","total","updown","up or down"]
```

### 5. 技术优化

#### 延迟优化
- 使用靠近 Polygon 节点的 VPS
- 减少 API 调用延迟
- WebSocket 实时数据优先

#### 自动化要点
- 遵守 API 限流
- 实现优雅降级
- 监控异常并报警

### 6. 行为准则

#### 不要
- 不要追涨杀跌
- 不要在冷却期内重复交易（1小时）
- 不要交易已持有市场的反向
- 不要忽视价差警告

#### 要做
- 记录每笔交易（Trade Journal）
- 定期分析胜率分布
- 根据数据调整策略权重
- 保持策略多样性

---

## 项目当前配置

### 价格过滤
```env
SIGNAL_TRADE_MIN_PRICE=0.80
SIGNAL_TRADE_MAX_PRICE=0.99
```

### 风险控制
```env
MAX_SLIPPAGE=0.01
MAX_SPREAD=0.03
MAX_TOTAL_POSITION=700
MAX_POSITION_PER_MARKET=30
```

### 交易金额
```env
SIGNAL_TRADE_AMOUNT_MIN=1.0
SIGNAL_TRADE_AMOUNT_MAX=5.0
```

---

## 优化建议

### 已完成
- ✅ Tiered Multipliers（分层乘数）
- ✅ Position Tracker（仓位追踪）
- ✅ Trade Aggregator（交易聚合）

### 待优化
- [ ] 动态调整价格区间阈值
- [ ] 根据胜率数据优化策略权重
- [ ] 实现更严格的滑点保护
- [ ] 添加更多市场质量指标

---

## 参考资源

1. [Polymarket Playbook by Nimrod Kamer](https://nnimrodd.medium.com/polymarket-playbook-5f53a2dd8ee3)
2. [Polymarket Strategies: 2026 Guide](https://cryptonews.com/cryptocurrency/polymarket-strategies/)
3. [Bitget Academy: Polymarket Trading Strategies](https://web3.bitget.com/en/academy/polymarket-trading-strategies-how-to-make-money-on-polymarket)
4. [QuantVPS: Automated Trading on Polymarket](https://www.quantvps.com/blog/automated-trading-polymarket)
5. [Laika AI: Top 10 Polymarket Trading Strategies](https://laikalabs.ai/prediction-markets/polymarket-trading-strategies)