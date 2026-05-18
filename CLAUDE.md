# 量化交易系统 — 项目说明

> 本文件为 Claude Code 在本仓库工作时的上下文摘要。完整设计见 `初步方案.md`。

---

## 一、项目背景

个人长期开发的多市场量化交易系统，定位为 **B- 档"个人长期工程"**：

- **不做玩具版**：不是 Jupyter 跑一段 demo 就完事。
- **不上机构重型版**：不堆 Kafka/Flink/微服务，避免维护负担压垮个人开发。
- **外围用成熟框架，核心交易规则自己掌控**：研究/存储/调度/连接用现成库，业务核心（Market Adapter、Strategy、Portfolio、Risk、OMS、Audit）自研。
- **第一版先跑通，再逐步扩展**：先美股 + 长桥 Paper + 简单动量跑通最小闭环，再扩三市场和多策略。

---

## 二、需求

| 维度 | 定位 |
| --- | --- |
| 资金规模 | 100 万人民币 |
| 开发模式 | 个人长期开发 |
| 策略频率 | 分钟（主要）、天 |
| 市场范围 | A 股 + 港股 + 美股 |
| 演化周期 | 5 年以上 |

**核心需求**：
1. 同一份策略代码能跑回测、模拟盘、实盘（只切数据源和执行器）。
2. 三市场（A/港/美）规则差异通过 Market Adapter 统一抽象。
3. 严格分层：策略只输出目标仓位，不直接下单；风控/OMS/券商接口职责单一不越界。
4. 必须能在五年内持续演化，不一次到位。

---

## 三、初步方案

### 3.1 总体架构

```
Market Adapter (横向枢纽：cn/hk/us，日历/费率/T+N/涨跌停/最小单位/汇率)
                ↕
DataHub → Factor Engine → Strategy Engine → Portfolio + Scheduler
                                                    ↓
                                Backtest / Paper Trading / Live Trading
                                                    ↓
                                        Risk Engine (事前/事中/事后)
                                                    ↓
                                              OMS (轻量订单管理)
                                                    ↓
                                Broker Adapter (长桥 / IBKR / 富途 / QMT)
                                                    ↓
                                        Monitor + Audit → Review
```

### 3.2 核心铁律

- 策略**只输出目标仓位**，绝不写 `broker.place_order(...)`。
- 策略**不知道**自己在回测/模拟/实盘，没有 `if is_live` 分支。
- **只有 Broker Adapter 触碰真实券商 API**，其他模块通过它访问。
- 回测必须模拟真实摩擦：手续费、印花税、滑点、T+1、涨跌停、停牌、最小单位、汇率。
- 风控三层：事前（下单前校验）/ 事中（运行中监控、熔断）/ 事后（对账）。

### 3.3 关键模块与技术选型

| 模块 | 自研程度 | 主要技术 |
| --- | --- | --- |
| DataHub | 半自研 | Parquet + DuckDB + PostgreSQL，pandas→polars |
| Market Adapter | **必须自研** | `exchange_calendars`，三市场子类 |
| Factor Engine | 半自研 | pandas-ta / TA-Lib，alphalens-reloaded |
| Strategy Engine | **必须自研** | 无现成框架，alpha 所在 |
| Portfolio | **必须自研轻量版** | 多策略目标合并 + 全局约束 |
| Scheduler | 不自研 | APScheduler 起步 |
| Backtest | 先框架后自研 | 阶段 1 `vectorbt` → 阶段 2 自研轻量（polars）→ 阶段 3 并存 |
| Paper Trading | 不自研为主 | 长桥 Paper / IBKR Paper / QMT 模拟 |
| Risk Engine | **必须自研** | 配置化阈值（`risk.yaml`） |
| OMS | **必须自研轻量状态机** | PostgreSQL 持久化，不上 vnpy |
| Broker Adapter | 半自研 | longport / ib_insync / futu-api / xtquant |
| Monitor + Audit | Monitor 不自研、Audit 必须自研 | loguru + Grafana + 飞书/TG 报警 |

### 3.4 券商演进路径

| 市场 | 起步 | 长期主力 |
| --- | --- | --- |
| 美股 | 长桥 | IBKR + ib_insync |
| 港股 | 长桥 | IBKR |
| A 股 | 国信证券 + QMT | QMT |

### 3.5 落地路线图（四阶段）

| 阶段 | 时长 | 目标 | 关键交付 |
| --- | --- | --- | --- |
| 1 | 0-3 月 | 长桥/美股闭环 | DataHub + Strategy + vectorbt + 长桥 Paper |
| 2 | 3-6 月 | 三市场扩展 | Market Adapter + A 股(QMT) + 港股 + 自研回测引擎 |
| 3 | 6-12 月 | 风控与小资金实盘 | 三层风控 + Audit + Monitor + 实盘 |
| 4 | 12 月+ | 平台化 | Portfolio 多策略 + 因子缓存 + Grafana + 期权 |

### 3.6 目标目录结构

```
quant-system/
├── data/            # raw / clean / features / factors / signals / positions
├── src/
│   ├── datahub/
│   ├── market/      # base.py / cn_stock.py / hk_stock.py / us_stock.py
│   ├── factors/
│   ├── strategies/
│   ├── portfolio/
│   ├── scheduler/
│   ├── backtest/
│   ├── paper/
│   ├── risk/
│   ├── oms/
│   ├── brokers/     # base.py / longbridge.py / ibkr.py / futu.py / qmt.py
│   ├── monitor/
│   └── audit/
├── scripts/         # download_data / run_backtest / run_paper / run_live / reconcile
├── notebooks/
├── tests/
└── logs/
```

---

## 四、当前仓库状态

- 仅有设计文档 `初步方案.md`，**尚未开始代码实现**。
- 后续按阶段 1 推进：先搭 `src/datahub/`、`src/strategies/`、`src/market/`、`src/brokers/longbridge.py` 的最小骨架。
