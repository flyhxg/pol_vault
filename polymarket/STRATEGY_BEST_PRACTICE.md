# Polymarket 交易策略最佳实践

> **最后更新**: 2026-03-20
> **版本**: 2.0
> **来源**: 项目代码分析 + 回测数据 + 生产运行经验

---

## 1. 核心原则

### 1.1 黄金法则

1. **高价格 = 高胜率**: 价格 ≥ 0.90 的信号胜率显著高于低价区间
2. **避免死亡区间**: 0.70-0.80 价格区间胜率仅 25%，必须自动过滤
3. **流动性为王**: 流动性不足的市场无法有效退出
4. **时间敏感**: 尾盘信号需要快速响应，延迟 = 损失

### 1.2 数据驱动决策

```
┌─────────────────────────────────────────────────────┐
│  信号流程: 数据 → 过滤 → 窗口确认 → 执行 → 监控   │
└─────────────────────────────────────────────────────┘

数据源优先级:
  1. WebSocket 实时价格 (最低延迟)
  2. REST API (fallback，延迟高)
  3. 缓存数据 (仅用于非实时分析)
```

---

## 2. 价格区间策略

### 2.1 价格区间胜率矩阵

| 价格区间 | 胜率 | EV | 风险等级 | 操作建议 |
|---------|------|-----|---------|---------|
| 0.95-1.00 | 89.5% | +$0.95 | 低 | ✅ 优先交易，金额可放大 |
| 0.90-0.95 | 75% | +$0.60 | 中低 | ✅ 正常交易 |
| 0.85-0.90 | 60% | +$0.20 | 中 | ⚠️ 需要额外确认 |
| 0.70-0.80 | 25% | -$0.39 | 高 | ❌ **死亡区间，必须过滤** |
| 0.50-0.70 | 35% | -$0.22 | 中高 | ⚠️ 仅限抄底策略 |
| <0.50 | 45% | -$0.10 | 中 | ⚠️ 反向操作可能有利 |

### 2.2 尾盘策略 (Endgame Sniper)

**定义**: 价格 ≥ 0.95 且剩余时间 ≤ 15 分钟

**执行规则**:
```python
# 尾盘信号优先级最高
if price >= 0.95 and time_to_expiry <= 15min:
    confidence_threshold = 0.50  # 降低置信度要求
    window_confirmations = 1      # 只需 1 个窗口
    amount_multiplier = 1.3       # 金额放大 30%
```

**关键点**:
- 延迟敏感：每秒延迟可能影响成交价格
- 使用 FOK 订单确保立即成交
- 避免在最后 2 分钟入场（流动性枯竭）

### 2.3 死亡区间过滤

**为什么 0.70-0.80 是死亡区间？**
- 胜率仅 25%，EV 为负
- 价格波动大，止损困难
- 容易被"拉盘"诱多

**代码实现**:
```python
# automaton_v2.py 内置过滤
if 0.70 <= yes_price <= 0.80:
    logger.info(f"[FILTER] 死亡区间: {yes_price:.2f}")
    return None  # 跳过信号
```

---

## 3. 订单流分析策略 (Smart Money)

### 3.1 信号类型矩阵

| 信号类型 | 触发条件 | 置信度 | 窗口需求 | 金额倍数 |
|---------|---------|--------|---------|---------|
| WHALE_SNIPER | 单笔 ≥$5,000 | 0.80 | 1 | 1.5x |
| CONSENSUS_FLOW | ≥3 人同向交易 | 0.70 | 2 | 1.2x |
| SWEEP_DETECTED | 短时间多笔同向 | 0.75 | 1 | 1.3x |
| MOMENTUM | 价格 + 净流入共振 | 0.65 | 2 | 1.0x |
| SMART_MONEY | 综合评分 ≥50 | 0.60 | 3 | 1.0x |

### 3.2 窗口确认机制

**窗口 = 时间段，用于确认信号真实性**

```python
WINDOW_DURATION = 30  # 每个窗口 30 秒

# 不同场景的窗口需求
WINDOW_REQUIREMENTS = {
    'endgame_price_gte_95': 1,    # 尾盘高价：1 窗口
    'endgame_price_gte_90': 2,    # 尾盘中价：2 窗口
    'normal_high_confidence': 2,  # 高置信度：2 窗口
    'normal': 3,                  # 普通信号：3 窗口
}

# 豁免条件（立即触发，无需等待）
EXEMPTION_RULES = [
    ('single_trade_gte', 5000),    # 单笔 ≥$5,000
    ('net_flow_gte', 10000),       # 净流入 ≥$10,000
    ('consensus_gte', 5),          # ≥5 人共识
]
```

### 3.3 Whale 白名单机制

**原理**: 只跟踪历史盈利的交易员

**当前白名单** (回测验证):
| 用户名 | 7天盈亏 | 胜率 | 跟踪优先级 |
|--------|--------|------|----------|
| theo5 | +$12,712 | 44% | 高 |
| BruceWayne77 | +$6,764 | 58% | 高 |
| allday24 | +$6,563 | 60% | 高 |
| Cinibengales | +$5,747 | 86% | 最高 |
| beachboy4 | +$4,235 | 78% | 高 |

**代码位置**: `polymarket_sdk/services/whale_scorer.py`

**更新频率**: 每周回测更新白名单

### 3.4 净流入阈值优化

**问题**: 默认净流入门槛 $1,500 过滤了太多信号

**建议分层**:
```python
# 尾盘降低门槛
if is_endgame:
    MIN_NET_FLOW = 500   # 尾盘只需 $500
else:
    MIN_NET_FLOW = 1500  # 正常 $1,500
```

---

## 4. 风险控制

### 4.1 交易参数配置

```bash
# 价格范围
SIGNAL_TRADE_MIN_PRICE=0.90    # 最低价格
SIGNAL_TRADE_MAX_PRICE=0.99    # 最高价格

# 置信度
MIN_CONFIDENCE=0.60            # 最低置信度

# 价差保护
MAX_SPREAD=0.03                # 最大价差 3%

# 流动性
SAFEGUARDS_MIN_LIQUIDITY=10000 # 最小流动性 $10,000

# 冷却时间
COOLDOWN_SECONDS=3600          # 同市场冷却 1 小时

# 金额限制
SIGNAL_TRADE_AMOUNT_MIN=1.0    # 最小 $1
SIGNAL_TRADE_AMOUNT_MAX=5.0    # 最大 $5
```

### 4.2 市场过滤规则

**关键词黑名单**:
```python
BLACKLIST_KEYWORDS = [
    'Up or Down',   # 纯赌博
    'Spread',       # 点差市场
    'O/U',          # Over/Under
    'Total',        # 总分市场
]
```

**市场类型过滤**:
```python
# 短期市场 (<6小时到期)
if time_to_expiry < 6 hours:
    skip_reason = "TOO_SHORT_TERM"

# 长期市场 (>30天)
if time_to_expiry > 30 days:
    skip_reason = "TOO_EARLY"
```

### 4.3 滑点保护

```python
# 滑点计算
spread = ask_price - bid_price
spread_pct = spread / mid_price

# 阈值
MAX_SPREAD_PCT = 0.03   # 3% 价差
WARNING_SPREAD = 0.015  # 1.5% 警告

# 检查
if spread_pct > MAX_SPREAD_PCT:
    reject_signal("价差过大")
```

---

## 5. 策略性能优化

### 5.1 扫描器性能问题

**问题**: `tail_strategy.scan()` 遍历所有市场，REST API 调用过多导致超时

**解决方案**:
```python
# 添加三重保护
self.max_markets_to_scan = 50     # 最多扫描 50 个市场
self.max_price_api_calls = 30     # 最多 30 次 API 调用
self.scan_timeout_seconds = 25    # 25 秒超时
```

**优先使用实时数据**:
```python
# 价格获取优先级
def _get_price_with_fallback(token_id):
    # 1. WebSocket 缓存 (最快)
    if realtime_service.has_price(token_id):
        return realtime_service.get_price(token_id)
    
    # 2. REST API (慢，但可靠)
    return await clob_api.get_best_prices(token_id)
```

### 5.2 信号去重与冷却

**问题**: 同一市场频繁触发信号

**解决**:
```python
# 市场冷却机制
COOLDOWN_SECONDS = 3600  # 1 小时

# 检查冷却
if market_id in recent_signals:
    if time_since_last < COOLDOWN_SECONDS:
        skip_signal("冷却中")
```

### 5.3 日志级别优化

**问题**: 日志过多影响性能

**建议**:
```python
# 生产环境
LOG_LEVEL=INFO

# 调试时
LOG_LEVEL=DEBUG

# 关键日志必须保留
logger.info(f"[{strategy}] 信号: {question} ({side} @ {price:.1%})")
logger.warning(f"[{strategy}] 过滤: {reason}")
```

---

## 6. 止盈止损策略

### 6.1 固定止损

```python
STOP_LOSS_PCT = -0.35  # -35% 止损
```

**执行逻辑**:
1. 记录入场价格
2. 实时监控当前价格
3. 当 P&L ≤ -35% 时卖出

### 6.2 移动止盈

```python
TRAILING_STOP_PCT = -0.10  # 从最高价回撤 10%

# 逻辑
if current_price > historical_high:
    historical_high = current_price
    
if current_price < historical_high * (1 - TRAILING_STOP_PCT):
    trigger_trailing_stop()
```

### 6.3 高价止盈

```python
TAKE_PROFIT_PRICE = 0.99  # 价格达到 $0.99 自动止盈

if current_price >= TAKE_PROFIT_PRICE:
    sell_all_shares()
```

---

## 7. 市场选择策略

### 7.1 类别胜率分析

| 类别 | 胜率 | 流动性 | 建议配置 |
|------|------|--------|---------|
| Sports | 80% | 高 | 优先交易 |
| Crypto | 67% | 高 | 可选 |
| Politics | 55% | 中 | 慎重 |
| Entertainment | 45% | 低 | 避免 |
| Other | 43% | 低 | 避免 |

### 7.2 流动性要求

```python
MIN_LIQUIDITY = 10000      # 最小流动性 $10,000
MIN_VOLUME_24H = 5000      # 最小 24h 成交量 $5,000
MAX_SPREAD_PCT = 0.03      # 最大价差 3%
```

---

## 8. 执行优化

### 8.1 订单类型选择

| 场景 | 订单类型 | 原因 |
|------|---------|------|
| 高确定性 + 流动性好 | FOK | 立即成交或取消 |
| 流动性差 | GTC | 挂单等待 |
| 大额交易 | 拆单 | 减少滑点 |
| 尾盘 | IOC | 立即成交剩余取消 |

### 8.2 金额动态调整

```python
def calculate_amount(signal):
    base_amount = 5.0
    
    # 尾盘加仓
    if is_endgame:
        base_amount *= 1.3
    
    # 大单加仓
    if net_flow >= 5000:
        base_amount *= 1.5
    
    # 高置信度加仓
    if confidence >= 0.80:
        base_amount *= 1.2
    
    return min(base_amount, MAX_AMOUNT)
```

---

## 9. 数据源管理

### 9.1 WebSocket vs REST API

| 特性 | WebSocket | REST API |
|------|----------|---------|
| 延迟 | <100ms | 200-500ms |
| 可靠性 | 需重连 | 高 |
| 资源占用 | 持续连接 | 按需请求 |
| 适用场景 | 实时交易 | 定时扫描 |

### 9.2 缓存策略

```python
# 市场缓存刷新周期
MARKET_CACHE_REFRESH_HOURS = 6

# 价格缓存过期时间
PRICE_CACHE_EXPIRE_SECONDS = 30
```

---

## 10. 监控与告警

### 10.1 进程监控

```bash
# crontab 每 5 分钟检查
*/5 * * * * python3 process_monitor.py
```

### 10.2 关键指标

| 指标 | 阈值 | 告警 |
|------|------|------|
| 信号执行率 | < 10% | ⚠️ 检查过滤条件 |
| 平均 P&L | < -$5 | ⚠️ 策略问题 |
| 进程运行时间 | > 24h | ℹ️ 建议重启 |
| 内存占用 | > 500MB | ⚠️ 内存泄漏 |

### 10.3 日志分析

```bash
# 查看信号统计
grep "发现.*信号" logs/*.log | tail -100

# 查看过滤原因
grep "FILTERED" logs/*.log | tail -100

# 查看错误
grep "ERROR\|Exception" logs/*.log | tail -50
```

---

## 11. 已知问题与解决方案

### 11.1 信号数量少

**原因分析**:
1. 净流入门槛太高 ($1,500)
2. 窗口确认太严格 (3 窗口)
3. Whale 白名单过时

**解决方案**:
```python
# 尾盘降低门槛
if is_endgame:
    MIN_NET_FLOW = 500

# 更新白名单
# 每周运行: python batch_backtest.py
```

### 11.2 扫描超时

**原因**: REST API 调用过多

**解决**:
```python
# 添加超时保护
self.scan_timeout_seconds = 25
self.max_price_api_calls = 30
```

### 11.3 价格获取失败

**原因**: 部分市场无订单簿

**解决**:
```python
# 跳过无价格市场
if bid is None or ask is None:
    stats['skip_price_fail'] += 1
    continue
```

---

## 12. 策略组合建议

### 12.1 推荐配置

```
主策略 (资金 70%):
  ├─ WHALE_SNIPER: 大单立即触发
  ├─ CONSENSUS_FLOW: 2 窗口确认
  └─ 尾盘信号: 1 窗口确认

辅助策略 (资金 20%):
  ├─ Tail Strategy: 趋势追踪
  └─ Trailing Stop: 抄底反弹

风险对冲 (资金 10%):
  └─ 低价反向操作 (谨慎)
```

### 12.2 避免的配置

- ❌ 在 0.70-0.80 价格区间交易
- ❌ 价差 > 3% 的市场
- ❌ 流动性 < $10,000 的市场
- ❌ 置信度 < 60% 的信号
- ❌ 长期市场 (>30 天) 过早入场
- ❌ 短期市场 (<6 小时) 延迟入场

---

## 13. 开发与调试

### 13.1 测试命令

```bash
# 单独测试策略
python3 skills_v2/tail_strategy.py

# 测试扫描器
python3 skills_integrator.py

# 运行主程序
python3 run_automaton_integrated.py

# 回测 Whale
python3 batch_backtest.py
```

### 13.2 调试模式

```bash
# 启用详细日志
LOG_LEVEL=DEBUG python3 run_automaton_integrated.py

# Dry run 模式
AUTO_TRADE_DRY_RUN=true python3 run_automaton_integrated.py
```

---

## 14. 参考资料

### 14.1 官方文档
- Polymarket Docs: https://docs.polymarket.com
- CLOB API: https://docs.polymarket.com/#clob-api
- Gamma API: https://gamma-api.polymarket.com/docs

### 14.2 开源项目
- CLOB Client: https://github.com/Polymarket/clob-client
- PyClobClient: https://github.com/Polymarket/py-clob-client
- Valory Trader: https://github.com/valory-xyz/trader

### 14.3 相关论文
- Prediction Market Efficiency
- Kelly Criterion in Prediction Markets
- Smart Money Tracking in Financial Markets

---

## 15. 更新日志

| 日期 | 版本 | 更新内容 |
|------|------|---------|
| 2026-03-20 | 2.0 | 添加性能优化、扫描超时、数据源管理 |
| 2026-03-19 | 1.0 | 初始版本 |

---

**维护者**: pol
**项目**: polymarket-agent