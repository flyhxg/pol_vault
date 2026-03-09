# Polymarket 信号监控系统

## 系统概述

Polymarket 自动交易信号监控系统，实时检测 Smart Money 交易和共识信号。

---

## 快速开始

### 1. 启动主系统

```bash
# 模拟模式（测试）
python run_automaton_integrated.py

# 真实交易模式
# 1. 配置 .env 文件（填写钱包私钥和地址）
# 2. 设置 AUTO_TRADE_DRY_RUN=false
# 3. python run_automaton_integrated.py
```

### 2. 启动 Web 界面

```bash
# FastAPI + Web 界面
python -m api.app

# 访问
http://localhost:8000
```

---

## Web 界面功能

### 信号监控面板

**特点：**
- 实时信号监控（WebSocket 推送）
- 深色主题设计
- 信号过滤漏斗分析
- 盈亏统计（胜率、ROI、总盈亏）
- 策略表现对比

**分析面板：**
- 总信号 / 已执行 / 已拒绝 / 执行率
- 过滤原因统计（死亡区间、关键词、价格区间、置信度、价差）
- 盈亏统计（胜率、ROI、已平仓、盈利/亏损次数）
- 策略对比（每个策略的交易数、胜率、盈亏）

---

## 配置

### 环境变量（.env 文件）

**模拟模式（默认）：**
```bash
AUTO_TRADE_DRY_RUN=true
AUTO_TRADE_ENABLED=false
```

**真实交易模式：**
```bash
# 钱包配置
PRIVATE_KEY=0x你的私钥
PROXY_WALLET_ADDRESS=0x你的钱包地址
CHAIN_ID=137

# 交易配置
AUTO_TRADE_DRY_RUN=false
AUTO_TRADE_ENABLED=true
SIGNAL_TRADE_AMOUNT_MIN=1.0
SIGNAL_TRADE_AMOUNT_MAX=5.0

# 风险控制
MIN_CONFIDENCE=0.60
BLOCKED_KEYWORDS=o/u,spread,total,updown,up or down
MAX_SPREAD=0.03
```

---

## 技术栈

- **后端：** Python, FastAPI
- **前端：** HTML, CSS, JavaScript
- **数据库：** SQLite
- **API：** Polymarket API
- **WebSocket：** 实时数据流

---

## 文件结构

```
polymarket-agent/
├── api/
│   ├── app.py                     # FastAPI 主应用
│   ├── websocket.py               # WebSocket 实时推送
│   └── routers/
│       ├── signals.py             # 信号 API
│       └── trades.py              # 交易历史 + 分析 API
├── web/templates/
│   └── signals.html               # Web 界面
├── run_automaton_integrated.py    # 主系统
├── automaton_v2.py                # 核心引擎
└── .env                           # 配置文件
```

---

## 安全提示

**真实交易前请确认：**

1. **钱包安全**
   - 私钥不要泄露
   - 钱包有足够 USDC
   - Polygon 主网需要 MATIC（gas）

2. **小额测试**
   - 建议先用 `SIGNAL_TRADE_AMOUNT_MAX=1.0` 测试
   - 观察 1-2 小时
   - 确认系统正常后再增加金额

3. **持续监控**
   - 观察信号质量
   - 检查交易记录
   - 及时调整参数

---

**Happy Trading!**