# Trade Journal 使用指南

## 📊 如何利用 Trade Journal 优化交易

### 第一步：定期查看交易表现（每周）

```bash
# 查看当前交易表现
python analyze_journal.py show
```

**查看指标：**
- ✅ 成功率 > 70% → 策略优秀
- ⚠️ 成功率 40-70% → 需要观察
- ❌ 成功率 < 40% → 策略有问题

### 第二步：生成优化建议（每周）

```bash
# 获取优化建议
python analyze_journal.py suggest
```

**输出示例：**
```
1️⃣ 价格区间分析:
   成功交易平均价格: 0.62
   失败交易平均价格: 0.81
   ✅ 建议: 优先交易价格 < 0.81 的市场
   配置: SIGNAL_TRADE_MAX_PRICE=0.80

2️⃣ 策略建议:
   ✅ copytrading: 表现优秀 (75.0%) → 增加权重
   ❌ mert_sniper: 表现较差 (35.0%) → 考虑禁用
```

**根据建议修改 .env：**
```bash
# 编辑配置文件
nano .env

# 应用建议
SIGNAL_TRADE_MAX_PRICE=0.80
ENABLE_MET_SNIPER=false  # 禁用表现差的策略
```

### 第三步：深度分析（每月）

```bash
# 导出所有数据
python analyze_journal.py export

# 使用 Excel/Pandas 分析
cd data/trade_journal
ls trades_analysis.csv
```

**在 Excel 中分析：**
1. 打开 `trades_analysis.csv`
2. 创建数据透视表
3. 分析：
   - 哪个市场类型最赚钱？
   - 哪个时间段胜率最高？
   - 哪种价格区间成功率最高？

### 第四步：根据数据调整参数

**示例场景 1：发现凌晨交易更赚钱**

```bash
# 分析发现：凌晨 2-4 点胜率 85%
# 其他时间胜率 55%

# 解决方案：只在凌晨运行
```

创建定时任务脚本：
```bash
# crontab -e
# 凌晨 2-6 点运行
0 2-6 * * * cd /path/to/polymarket-agent && python run_automaton_integrated.py
0 6 * * * pkill -f run_automaton_integrated  # 早上 6 点停止
```

**示例场景 2：价格太高的交易容易失败**

```bash
# 分析发现：
# - 价格 < 0.60: 成功率 75%
# - 价格 > 0.80: 成功率 40%

# 调整配置
nano .env
SIGNAL_TRADE_MIN_PRICE=0.30
SIGNAL_TRADE_MAX_PRICE=0.70
```

**示例场景 3：某个策略一直亏钱**

```bash
# 查看 copytrading 表现
python analyze_journal.py show | grep copytrading

# 如果成功率 < 40%，禁用它
nano .env
ENABLE_COPYTRADING=false
```

### 第五步：建立分析习惯

**每日（5分钟）：**
```bash
# 快速查看今天交易
python analyze_journal.py show | grep "总交易数"
```

**每周（30分钟）：**
```bash
# 1. 查看完整报告
python analyze_journal.py all

# 2. 导出数据做深度分析
python analyze_journal.py export

# 3. 根据建议调整参数
nano .env
```

**每月（2小时）：**
1. 导出数据到 Excel
2. 创建图表分析趋势
3. 测试新参数配置
4. 更新交易策略

---

## 🔧 实用技巧

### 技巧 1：对比不同时期的表现

```bash
# 备份数据
cp -r data/trade_journal data/trade_journal_backup_$(date +%Y%m%d)

# 运行一周后对比
diff data/trade_journal_backup_20260301/summary.json \
     data/trade_journal/summary.json
```

### 技巧 2：A/B 测试

**测试新参数：**
```bash
# 第 1 周：使用当前参数
# 记录表现
python analyze_journal.py export

# 第 2 周：修改参数
nano .env
SIGNAL_TRADE_MAX_PRICE=0.70  # 从 0.99 改为 0.70

# 第 2 周结束后对比
python analyze_journal.py show
```

### 技巧 3：识别"运气交易"

查看单个市场：
```python
import json
from pathlib import Path

# 读取校准数据
with open('data/trade_journal/calibration.json') as f:
    data = json.load(f)

# 找出交易次数 > 3 的市场
for cid, cal in data.items():
    if len(cal['trades']) > 3:
        print(f"{cal['question'][:50]}...")
        print(f"  交易次数: {len(cal['trades'])}")
        print(f"  价格变化: {cal['price_end'] - cal['price_start']:.2f}")
```

---

## 📈 优化循环

```
┌─────────────────────┐
│  1. 运行交易系统     │
│     (1周)           │
└──────────┬──────────┘
           │
           v
┌─────────────────────┐
│  2. 分析 Trade Journal│
│  python analyze_journal │
└──────────┬──────────┘
           │
           v
┌─────────────────────┐
│  3. 发现问题         │
│  - 成功率下降？      │
│  - 某策略亏损？      │
│  - 价格区间不好？    │
└──────────┬──────────┘
           │
           v
┌─────────────────────┐
│  4. 调整参数         │
│  nano .env          │
└──────────┬──────────┘
           │
           v
┌─────────────────────┐
│  5. 重新测试         │
│  回到步骤 1         │
└─────────────────────┘
```

---

## 💡 高级用法

### 1. 自动化分析报告

创建 `scripts/weekly_report.sh`：
```bash
#!/bin/bash
cd /path/to/polymarket-agent

# 生成报告
python analyze_journal.py all

# 发送到 Telegram（如果配置了）
curl -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
  -d "chat_id=$TELEGRAM_CHAT_ID" \
  -d "text=📊 周报已生成: $(cat data/trade_journal/latest_report.json | jq '.summary')"

# 备份数据
tar -czf backup_$(date +%Y%m%d).tar.gz data/trade_journal/
```

添加到 crontab：
```bash
# 每周日早上 9 点生成周报
0 9 * * 0 /path/to/weekly_report.sh
```

### 2. 策略回测

使用 Trade Journal 数据回测新策略：
```python
# backtest_strategy.py
import json
import pandas as pd

# 加载历史交易
df = pd.read_json('data/trade_journal/trades.jsonl', lines=True)

# 测试假设：只在价格 < 0.70 时交易
filtered = df[df['yes_price'] < 0.70]
success_rate = filtered['success'].mean()

print(f"回测结果: {success_rate:.1%}")
```

### 3. 实时监控

在系统运行时实时查看表现：
```bash
# 终端 1：运行主系统
python run_automaton_integrated.py

# 终端 2：实时监控
watch -n 60 'python analyze_journal.py show'
```

---

## 🎯 目标设定

**第 1 个月目标：**
- ✅ 完成至少 100 笔交易
- ✅ 成功率稳定在 50% 以上
- ✅ 识别 2 个盈利策略

**第 3 个月目标：**
- ✅ 总交易 > 500 笔
- ✅ 成功率 > 60%
- ✅ 找出最佳交易时间段

**第 6 个月目标：**
- ✅ 稳定盈利
- ✅ 策略完全优化
- ✅ 自动化运行

---

**记住：Trade Journal 是你的交易日记，坚持记录和分析是成功的关键！** 📖
