# Journal-Based Bandit 使用指南

## 🎯 功能说明

Journal-Based Bandit 是一个智能策略选择器，它会：

1. **从 Trade Journal 读取历史数据** - 了解哪些策略赚钱
2. **自动调整策略权重** - 给表现好的策略更高优先级
3. **定期重新校准** - 每 24 小时根据最新数据更新

## 📊 工作原理

```
┌─────────────────────────────────────────────┐
│  Trade Journal (历史交易数据)                │
│  - copytrading: 75% 胜率                    │
│  - mert_sniper: 35% 胜率                    │
│  - signal_sniper: 60% 胜率                  │
└──────────────┬──────────────────────────────┘
               │
               v
┌─────────────────────────────────────────────┐
│  Journal-Based Bandit                       │
│  1. 计算每个策略的真实奖励                   │
│  2. 使用历史数据初始化 UCB 分数              │
│  3. 结合当前数据动态调整                    │
└──────────────┬──────────────────────────────┘
               │
               v
┌─────────────────────────────────────────────┐
│  AutomatonV2 策略选择                       │
│  - 优先选择历史表现好的策略                  │
│  - 但仍然探索新策略（UCB 算法）              │
└─────────────────────────────────────────────┘
```

## 🚀 如何使用

### 方法 1：自动启动（推荐）

重启主系统时会自动启动：

```bash
python run_automaton_integrated.py
```

日志中会看到：
```
[JournalBandit] 初始化完成
[JournalBandit] 开始从历史数据校准...
[JournalBandit] ✅ 已校准 3 个策略
🏆 策略表现排名（基于历史数据）
  🥇 第1名: copytrading            - 75.0分
  🥈 第2名: signal_sniper          - 60.0分
  🥉 第3名: weather_trader         - 55.0分
[AutomatonV2] ✅ Journal-Based Bandit 已启动
```

### 方法 2：手动校准

```bash
# 立即执行一次校准
python bandit_journal.py calibrate

# 持续监控模式（每小时重新校准）
python bandit_journal.py watch

# 导出校准数据
python bandit_journal.py export
```

### 方法 3：查看策略排名

```bash
# 查看基于历史数据的策略表现排名
python bandit_journal.py calibrate
```

输出示例：
```
🏆 策略表现排名（基于历史数据）
================================================================================
🥇 第1名: copytrading              - 75.0分
     交易数: 45
     成功率: 75.0%
     平均成本: $5.00

🥈 第2名: signal_sniper            - 60.0分
     交易数: 30
     成功率: 60.0%
     平均成本: $5.00

🥉 第3名: weather_trader           - 55.0分
     交易数: 12
     成功率: 55.0%
     平均成本: $5.00
```

## 📈 实际效果

### 之前（没有 Journal-Based Bandit）：

```
策略选择：完全随机/平均分配
- copytrading: 30% 选择率（但实际上胜率 75%）
- mert_sniper:  30% 选择率（但实际上胜率 35%）
- signal_sniper: 40% 选择率（但实际上胜率 60%）

结果：浪费资源在亏损策略上
```

### 之后（有 Journal-Based Bandit）：

```
策略选择：基于历史表现 + UCB 探索
- copytrading: 60% 选择率 ✅（历史胜率 75%，优先使用）
- signal_sniper: 35% 选择率 ✅（历史胜率 60%，适度使用）
- mert_sniper:  5% 选择率  ⚠️（历史胜率 35%，偶尔探索）

结果：更多使用赚钱的策略！
```

## 📅 使用时间表

**每日（自动）：**
- Journal-Based Bandit 在后台运行
- 每小时检查是否需要重新校准
- 自动更新策略权重

**每周（手动建议）：**
```bash
# 查看当前策略排名
python bandit_journal.py calibrate

# 分析 Trade Journal
python analyze_journal.py show

# 根据数据调整 .env 配置
```

**每月（深度分析）：**
```bash
# 导出所有数据
python bandit_journal.py export
python analyze_journal.py export

# 在 Excel 中深度分析
# - 哪些策略在特定时间段表现更好？
# - 价格区间对胜率的影响？
# - 是否需要调整参数？
```

## 💡 优化建议

### 建议 1：根据排名禁用亏损策略

如果某个策略连续 2 个月胜率 < 40%：

```bash
# 查看排名
python bandit_journal.py calibrate

# 假设发现 mert_sniper 只有 35% 胜率
# 编辑 skills_integrator.py
nano skills_integrator.py

# 修改 enable_skills
'mert_sniper': False,  # 暂时禁用
```

### 建议 2：给优秀策略增加资金

如果某个策略胜率 > 70%：

```bash
# 增加该策略的默认交易金额
# 编辑 skills_integrator.py
nano skills_integrator.py

# 找到对应策略初始化
self.mert_sniper = MertSniperV2(
    ...
    default_amount=10.0,  # 从 $5 增加到 $10
)
```

### 建议 3：定期重新校准

Journal-Based Bandit 默认每 24 小时自动校准一次。

如果交易频繁，可以缩短间隔：

```python
# bandit_journal.py
journal_bandit = JournalBasedBandit(
    ...
    calibration_interval_hours=12  # 改为 12 小时
)
```

## 🔧 故障排除

### 问题：校准失败

```
[JournalBandit] 没有历史数据，使用默认值
```

**解决：**
需要先运行系统产生一些交易记录。等待至少 5 笔交易后再校准。

### 问题：策略没有足够交易

```
⚠️  mert_sniper: 交易数不足 (3 < 5)
```

**解决：**
等待更多交易，或者降低 `min_trades_for_calibration`：

```python
# bandit_journal.py
journal_bandit = JournalBasedBandit(
    ...
    min_trades_for_calibration=3  # 降低到 3 笔
)
```

### 问题：某些策略没有被校准

可能原因：
1. 策略从未被使用过
2. 交易全部失败（success=False）
3. Trade Journal 数据损坏

**检查：**
```bash
# 查看 Trade Journal 原始数据
python -c "
from skills_v2.trade_journal_v2 import TradeJournalV2
journal = TradeJournalV2(sdk=None)
journal._load_data()
for t in journal.trades:
    print(f'{t.strategy}: {t.success}')
"
```

## 📊 与其他组件配合

### 1. 配合 Trade Journal 分析

```bash
# 查看策略表现
python bandit_journal.py calibrate

# 查看详细分析
python analyze_journal.py show

# 获取优化建议
python analyze_journal.py suggest
```

### 2. 配合参数管理器

ParamManager 会定期自动调整参数，Journal-Based Bandit 提供历史数据支持。

### 3. 配合 Bandit 算法

- **Journal-Based Bandit**：长期、历史数据的校准
- **标准 Bandit**：短期、实时反馈的学习

两者结合效果最好！

---

**总结：Journal-Based Bandit 让系统"吃一堑长一智"，从历史经验中学习，避免重复犯错！** 🧠
