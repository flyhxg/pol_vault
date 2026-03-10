# Polymarket 交易优化方案

## 当前架构

### 1. Smart Money (WebSocket 实时)
- **数据源**: Polymarket WebSocket 交易流
- **分析逻辑**: 净流入检测 (≥$5000)
- **触发条件**: 3笔以上大额交易 + 净流入
- **模式**: top_n=0 (开放模式，监控所有交易员)

### 2. Copytrading V2 (Data API 定期)
- **数据源**: Data API 历史交易查询
- **分析逻辑**: 共识检测 (≥5人)
- **触发条件**: Top 200 交易员中 ≥5 人交易同一市场
- **扫描频率**: 30秒

---

## 日志分析发现

### 问题 1: 低价区交易胜率 0%
```
低价区 (<$0.60): 胜率 0.0%  (0/15)
中价区 ($0.60-0.80): 胜率 0.0%  (0/4)
高价区 (>$0.80): 胜率 87.5% (14/16)
```
**优化**: 提高 min_price 到 0.80

### 问题 2: 交易金额影响
```
小额 ($1-2): 未统计
中额 ($3-4): 胜率 0.0%  (0/15)
大额 ($5+): 胜率 70.0% (14/20)
```
**优化**: 提高 min_amount 到 5.0

### 问题 3: 时段差异
```
12:00, 21:00: 胜率 100%
13:00-14:00, 18:00-20:00: 胜率 0%
```
**优化**: 添加时段过滤

### 问题 4: 类别差异
```
crypto: 66.7% 胜率 ✅
politics: 50.0% 胜率
sports: 38.9% 胜率 ⚠️
other: 30.0% 胜率 ❌
```
**优化**: 提高 sports 门槛，暂停 other

---

## 优化方案

### 1. Smart Money 优化
```python
# 当前
min_net_flow=5000
min_trader_count=3
min_price=0.50

# 优化后
min_net_flow=8000        # 提高门槛
min_trader_count=5       # 更严格
min_price=0.80          # 只交易高价区
enable_strict_sports=true  # Sports 2倍门槛
```

### 2. Copytrading V2 优化
```python
# 当前
consensus_threshold=5
time_window_minutes=1

# 优化后
consensus_threshold=3    # 降低门槛，增加信号量
max_consensus=0.70      # 避免过度共识（70-85%最佳）
time_window_minutes=2   # 延长窗口
```

### 3. 时段过滤
```python
# 添加时段检查
ALLOWED_HOURS=0,1,2,3,4,5,10,12,15,21,23  # 只在这些时段交易
BLOCKED_HOURS=13,14,18,19,20  # 跳过这些时段
```

### 4. Trade Journal 反馈
```python
# 每10笔交易后分析
if trade_count % 10 == 0:
    analyze_and_adjust_thresholds()
```

---

## 实施步骤

1. [x] 提高 min_price 到 0.80
2. [x] 启用 strict_sports
3. [ ] 调整 Copytrading 共识阈值
4. [ ] 添加时段过滤
5. [ ] 增加 Trade Journal 自动分析

---

*更新于 2026-03-10*