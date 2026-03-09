# Polymarket Signals - Polymarket 自动交易系统

**基于 Smart Money 跟踪策略的自动化交易系统**

---

## 项目简介

Polymarket Signals 是一个实时跟踪 Polymarket 高胜率交易员并自动跟单的交易系统。系统有两个独立的信号源：

1. **Smart Money Service**：通过 WebSocket 实时接收所有交易活动（36笔/秒），检测"大资金持续流入"模式（单笔 ≥ $100，60秒内净流入 ≥ $1000）

2. **Copytrading V2**：从 Data API 获取 Top 50 盈利交易员列表，定期扫描这些交易员的持仓并执行跟单交易

### 核心特性

- ✅ **Smart Money 大资金检测**：单笔交易 ≥ $200，60秒窗口内净流入 ≥ $1000（不依赖交易员身份）
- ✅ **Top 50 盈利交易员持仓跟单**：定期扫描 Top 50 盈利交易员持仓并跟单
- ✅ **6 个技能模块**：Mert Sniper、Signal Sniper、Copytrading、Weather Trader、Fast Loop、Trade Journal
- ✅ **统一 Safeguards 检查**：市场状态、价格区间、流动性、价差、滑点等多层验证
- ✅ **完全自动化**：从信号检测到交易执行
- ✅ **Dry Run 模式**：测试模式，不执行实际交易

---

## 系统架构

```
Smart Money Service (独立信号源)
        ↓
WebSocket (Realtime Service) - 实时接收所有交易活动 (36笔/秒)
        ↓
单笔交易过滤 (≥ $100)
        ↓
大资金持续流入检测 (60秒窗口内净流入 ≥ $1000)
        ↓
直接生成信号 → Automaton V2


Data API (Leaderboard)
        ↓
Skills V2 (6个技能模块)
  ├─ Copytrading V2        - 获取 Top 50 盈利交易员，扫描持仓跟单
  ├─ Mert Sniper V2        - 尾盘机会（8分钟窗口）
  ├─ Signal Sniper V2      - RSS 新闻狙击
  ├─ Weather Trader V2     - 天气事件交易
  ├─ Fast Loop V2          - BTC/ETH/SOL 快速市场
  └─ Trade Journal V2      - 交易日志
        ↓
Skills Integrator (统一扫描)
        ↓
生成信号 → Automaton V2


Automaton V2 (决策引擎)
  ├─ 价格验证
  ├─ 置信度过滤
  ├─ 持仓限制
  └─ 执行交易
        ↓
Polymarket SDK
```

---

## 实际启用的技能（6个）

| 技能 | 状态 | 说明 |
|------|------|------|
| Mert Sniper V2 | ✅ 启用 | 尾盘机会（8分钟窗口） |
| Signal Sniper V2 | ✅ 启用 | RSS 新闻狙击 |
| Copytrading V2 | ✅ 启用 | Top 50 盈利交易员持仓跟单 |
| Weather Trader V2 | ✅ 启用 | 天气事件交易 |
| Fast Loop V2 | ✅ 启用 | BTC/ETH/SOL 快速市场 |
| Trade Journal V2 | ✅ 启用 | 交易日志（被动记录） |
| AI Divergence V2 | ❌ 禁用 | AI 分歧度检测（暂时禁用） |

---

## 目录结构

```
polymarket-signals/
├── README.md                      # 本文件
├── .env.example                   # 环境变量模板
├── requirements.txt               # Python 依赖
│
├── api/                           # API 服务器 + Web 界面
│   ├── app.py                     # FastAPI 主应用
│   ├── websocket.py               # WebSocket 实时推送
│   └── routers/
│       ├── signals.py             # 信号 API
│       └── trades.py              # 交易历史 + 分析 API
│
├── web/                           # Web 前端
│   └── templates/
│       └── signals.html           # 信号监控界面（含分析面板）
│
├── data/                          # 数据目录
│   ├── logs/                     # 日志文件
│   └── trade_journal/            # 交易日志
│
├── polymarket_sdk/               # Polymarket SDK
│   ├── clients/                  # API 客户端
│   ├── services/                 # 业务服务
│   │   ├── smart_money_service_v2.py
│   │   ├── position_monitor/
│   │   └── signal_recorder.py
│   └── utils/                    # 工具函数
│
├── skills_v2/                    # 技能模块
│   ├── mert_sniper_v2.py         # 尾盘机会
│   ├── signal_sniper_v2.py       # RSS 新闻狙击
│   ├── copytrading_v2.py         # Top 50 盈利交易员跟单
│   ├── weather_trader_v2.py      # 天气事件交易
│   ├── fast_loop_v2.py           # BTC/ETH/SOL 快速市场
│   ├── trade_journal_v2.py       # 交易日志（被动记录）
│   └── state/                    # 技能状态
│
├── automaton_v2.py               # Automaton V2 核心引擎
├── skills_integrator.py          # 技能集成器
└── run_automaton_integrated.py   # 主运行脚本 ⭐
```

---

## 快速开始

### 1. 环境要求

- Python 3.10+
- Windows/Linux/Mac

### 2. 安装依赖

```bash
# 克隆项目
git clone https://github.com/West-Garden/polymarket-agent.git
cd polymarket-signals

# 创建虚拟环境
python -m venv venv

# 激活虚拟环境
# Windows:
venv\Scripts\activate
# Linux/Mac:
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt
```

### 3. 配置环境变量

```bash
# 复制环境变量模板
cp .env.example .env

# 编辑 .env 文件，填写必填配置
```

**必填配置：**

```bash
# Polymarket API（从 https://polymarket.com/ 获取）
POLYMARKET_API_KEY=your_api_key_here
POLYMARKET_API_SECRET=your_api_secret_here
POLYMARKET_API_PASSPHRASE=your_api_passphrase_here

# 钱包配置
PRIVATE_KEY=0xyour_private_key_here
PROXY_WALLET_ADDRESS=0xyour_proxy_wallet_address_here
CHAIN_ID=137

# 交易配置
AUTO_TRADE_DRY_RUN=true              # 测试模式（true=不执行交易，false=真实交易）
SIGNAL_TRADE_AMOUNT_MIN=1.0         # 最小交易金额（美元）
SIGNAL_TRADE_AMOUNT_MAX=5.0         # 最大交易金额（美元）
```

**可选配置（对应技能需要时启用）：**

```bash
# Weather Trader 技能
OPENWEATHER_API_KEY=your_openweather_key
```

### 4. 启动系统

**启动交易机器人：**
```bash
python run_automaton_integrated.py
```

**启动 Web 界面（另一个终端）：**
```bash
python -m api.app
```

访问：
- http://localhost:8000 → Web 界面
- http://localhost:8000/docs → API 文档

**Web 界面功能：**
- 实时信号监控（WebSocket 推送）
- 信号过滤漏斗分析
- 盈亏统计（胜率、ROI、总盈亏）
- 策略表现对比

---

## 实际运行情况（基于真实数据）

### 信号源 1：Smart Money Service

根据实际运行日志：

```
数据源: WebSocket 实时交易流
监控范围: 所有交易活动
策略: 大资金持续流入检测
  - 单笔交易过滤: ≥ $200（提高阈值，减少噪音）
  - 分析窗口: 60 秒
  - 净流入阈值: ≥ $500-$5000（动态，根据价格阶段）
  - 最少交易笔数: ≥ 2-3 笔（动态）

接收交易: 27,000+ 笔
交易速度: 36 笔/秒
单笔过滤: 大部分被过滤 (< $200)
检测到候选: 10 个
过滤掉: 26,990 个 (99.96%)
维护缓冲: 5 个窗口
```

### 信号源 2：Copytrading V2

```
数据源: Data API Leaderboard
白名单: Top 50 盈利交易员
策略: 定期扫描持仓
Top 50 交易员: 100% 盈利，平均 PnL $211,119
扫描结果: 0 个候选（市场流动性太差）
```

### 技能信号生成情况

| 技能 | 实际信号量 | 原因分析 |
|------|-----------|---------|
| Mert Sniper V2 | 0 | 当前无 8 分钟内到期的市场 |
| Signal Sniper V2 | 0 | RSS 新闻与市场匹配度低 |
| Copytrading V2 | 0 候选 | Smart Money 交易的市场流动性太差 |
| Weather Trader V2 | 0 | 当前无极端天气事件 |
| Fast Loop V2 | 0 | BTC/ETH/SOL 价格变动 < 2% |
| Trade Journal V2 | N/A | 仅记录，不生成信号 |

### 为什么没有信号？

**Smart Money Service 的过滤条件（非常严格）：**
- ❌ 置信度不足 (< 60%)
- ❌ 价格超出范围 (< $0.15 或 > $0.99)
- ❌ 死亡区间 [0.70-0.80]（胜率仅 25%）
- ❌ 包含屏蔽关键词（up or down, spread 等）
- ❌ 价差太大 (> 3%)
- ❌ 流动性不足 (< $10,000)
- ❌ 时间太长 (> 24 小时)

**策略说明：**
Smart Money Service 不依赖交易员身份（白名单），而是检测"大资金持续流入"行为：
1. 单笔交易 ≥ $100（过滤散户小额交易）
2. 60 秒窗口内净流入 ≥ $1000（确保是大资金，不是散户）
3. 至少 3 笔交易（确保持续流入，不是单次大单）

**结果：**
- Smart Money 检测到的 10 个候选
- 全部被后续过滤条件拦截
- Copytrading V2 收集到 0 个候选信号

**这是正常的：**
- Smart Money 交易的市场流动性差
- Spread 往往超过 3%
- 不适合小资金跟单

---

## 配置说明

### Safeguards 配置（统一市场质量检查）

```bash
# 时间限制
SAFEGUARDS_MAX_TIME_TO_EXPIRY=24.0  # 最大剩余时间（小时）

# 价格区间（尾盘策略：高价=趋势已确立=安全入场）
SAFEGUARDS_MIN_PRICE=0.50  # 最低价格（过滤低价不确定信号，趋势更强）
SAFEGUARDS_MAX_PRICE=0.99  # 最高价格（允许高价入场，尾盘趋势确认）
SAFEGUARDS_ENABLE_PRICE_FILTER=true  # 是否启用价格过滤

# 价差和滑点
SAFEGUARDS_MAX_SPREAD=0.03  # 最大价差（3%）
SAFEGUARDS_MAX_SLIPPAGE=0.02  # 最大滑点（2%）
SAFEGUARDS_HIGH_SLIPPAGE=0.05  # 高滑点阈值（5%）
SAFEGUARDS_MODERATE_SLIPPAGE=0.02  # 中等滑点阈值（2%）

# 流动性
SAFEGUARDS_MIN_LIQUIDITY=10000.0  # 最小流动性（美元）
SAFEGUARDS_MIN_DEPTH=100.0  # 最小订单簿深度（tokens，前5档）

# 最小份额
SAFEGUARDS_MIN_SHARES=5  # 每单最小份额
```

### 自动交易配置

```bash
# 测试模式（不执行实际交易）
AUTO_TRADE_DRY_RUN=true

# 交易金额（美元）
SIGNAL_TRADE_AMOUNT_MIN=1.0
SIGNAL_TRADE_AMOUNT_MAX=5.0
```

### 风险控制

```bash
# 最小置信度
MIN_CONFIDENCE=0.60

# 死亡区间过滤（价格 [0.70-0.80] 胜率仅 25%）
# 已内置在 automaton_v2.py 中

# 关键词过滤（屏蔽 Up or Down, Spread 等市场）
BLOCKED_KEYWORDS=o/u,spread,total,updown,up or down

# 最大价差
MAX_SPREAD=0.03
```

---

## 监控运行

### 查看日志

```bash
# 实时日志（最新日志文件）
tail -f data/logs/automaton_integrated/automaton_integrated_*.log
```

### 系统统计

日志每分钟输出统计：

```
[STATUS] 系统运行中 | 运行时间: X 分钟
   信号接收: N | 执行交易: 0 | 当前持仓: 40
   Skills 信号: 0
```

### Web 监控界面

启动：`python -m api.app`

访问：http://localhost:8000

**功能：**
- 实时信号展示（WebSocket 推送）
- 信号过滤漏斗（死亡区间、关键词、价格区间、置信度、价差）
- 盈亏统计（胜率、ROI、总盈亏、已平仓数）
- 策略表现对比（每个策略的交易数、胜率、盈亏）
- 信号源标签和置信度显示
- 执行/拒绝状态

---

## 技术栈

- **Python 3.10+**
- **AsyncIO** - 异步编程
- **FastAPI** - Web 框架（可选）
- **WebSocket** - 实时市场订单簿和价格数据
- **Polymarket SDK** - 自研 SDK

---

## 风险提示

1. **本系统当前 0 信号生成**
   - Smart Money 过滤率 99.96%
   - 所有技能暂时无信号
   - 这是正常的（市场条件不符合）

2. **即使有信号，也不保证盈利**
   - Smart Money 交易的市场流动性差
   - Spread 往往很大（>3%）
   - 跟单可能亏损

3. **建议先使用 Dry Run 模式**
   - 设置 `AUTO_TRADE_DRY_RUN=true`
   - 观察系统运行情况
   - 理解信号质量后再考虑真实交易

---

## 免责声明

本软件仅供学习和研究使用。预测市场交易存在风险，使用本软件的一切损失由使用者自行承担。

---

**最后更新**: 2026-03-07
**仓库**: https://github.com/West-Garden/polymarket-agent
