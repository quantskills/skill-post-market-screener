# Post-Market Screener 操作手册

> v2.0 | 2026-06-30 | 同花顺资金流向主路径 + 双因子模式已生产验证

---

## 1. 环境要求

| 组件 | 最低版本 | 说明 |
|---|---|---|
| Python | 3.10+ | 建议 3.11 |
| pip | 23.0+ | — |
| Pandadata 账号 | — | K线 + 股票信息数据 |
| DeepSeek API Key | — | LLM 分析必需 |

### 1.1 安装依赖

```bash
cd skill-post-market-screener
pip install -r requirements.txt
```

依赖清单：

```
panda_data==1.0.6      # Pandadata API SDK
pandas>=2.0.0           # 数据处理
numpy>=1.22,<2.0        # 数值计算
pyyaml>=6.0             # 配置解析
python-dotenv>=1.0.0    # 环境变量加载
anthropic>=0.39.0       # LLM API 客户端（DeepSeek 兼容）
akshare>=1.18.0         # 同花顺 + 东方财富资金流向数据
requests>=2.31.0        # HTTP 会话管理（东方财富直连 API）
urllib3>=2.0.0          # HTTP 重试适配器
```

### 1.2 配置凭证

**Pandadata（K线 + 股票信息）：**

方式 A — 环境变量（推荐）：
```bash
export DEFAULT_USERNAME="86+你的手机号"
export DEFAULT_PASSWORD="你的密码"
```

方式 B — `config.json`：
```json
{
  "pandadata": {
    "base_url": "http://pandadata.pandaaiquant.com",
    "username": "86+你的手机号",
    "password": "你的密码"
  }
}
```

**DeepSeek API（LLM 分析）：**

```bash
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="sk-你的DeepSeek密钥"
export ANTHROPIC_MODEL="deepseek-v4-pro"
```

---

## 2. 运行模式

### 2.1 命令总览

| 命令 | 说明 |
|---|---|
| `python run.py` | **标准生产模式**：Pandadata K线 + 同花顺资金流 + LLM 分析 |
| `python run.py --date 20260630` | 指定交易日扫描 |
| `python run.py --no-flow` | 纯形态扫描（跳过资金过滤） |
| `python run.py --top-n 10` | 报告只显示 Top 10 只个股 |
| `python run.py --config my_config.json` | 使用自定义配置文件 |
| `python scripts/analyze_weights.py --data cache/ --json` | **回测权重校准**：IC分析 + 权重优化 |

### 2.2 权重校准（回测验证闭环）

检测器权重不是固定经验值——通过历史数据回测定期校准，形成"信号 → 回测 → 优化权重 → 生产"的闭环。

```bash
# 单次回测（需 ≥60 天缓存数据）
python scripts/analyze_weights.py --data cache/ --json

# 滚动窗口稳定性分析
python scripts/analyze_weights.py --data cache/ --rolling

# 输出到 config.json
python scripts/analyze_weights.py --data cache/ --json > /dev/null && echo "Weights saved to config.json"

# 采样加速（1000 只代表样本）
python scripts/analyze_weights.py --data cache/ --sample 1000 --json
```

**回测指标说明：**

| 指标 | 说明 | 判断标准 |
|---|---|---|
| IC(5d) | 信号强度 vs 5 日未来收益的 Spearman 秩相关 | > 0.02 优秀，< -0.02 负向 |
| Hit Rate | 信号触发后 5 日正收益占比 | > 50% 优秀，< 45% 需降权 |
| IC IR | IC 均值 / IC 标准差（bootstrap） | > 0.3 稳定 |
| 信号数 | 检测器触发次数 | < 10 不可靠，≥ 1000 统计有效 |

权重调整后写入 `config.json` → `detector_weights`，流水线 `apply_weights_from_config()` 自动加载。建议每季度重跑一次。

> 最近一次回测（2026-06-30）：1000 只样本、233,894 条记录。放量突破(4→3)、RSI超卖(2→1)。启明星 IC 最优。

### 2.3 完整命令行参数

```
python run.py [OPTIONS]

Options:
  --date YYYYMMDD   目标交易日期（默认：最新交易日）
  --config PATH     配置文件路径（默认：./config.json）
  --no-flow         跳过资金流向过滤（纯形态扫描）
  --top-n N         报告中的个股数量（默认：config.json scan.top_n，回退 20）
```

### 2.4 数据来源

| 数据类别 | 来源 | 说明 |
|---|---|---|
| K线数据 | Pandadata `get_stock_daily` | 全市场 ~5186 只，120 天历史 |
| 资金流向 | **同花顺** `stock_fund_flow_individual` (10jqka) | 主力净额/成交额，独立于东方财富 |
| 股票信息 | Pandadata `get_stock_detail` / `get_trade_list` | 行业分类、市值、上市状态 |
| LLM分析 | DeepSeek API (Anthropic 兼容) | 逐只个股技术+资金综合分析 |

---

## 3. 资金流向容灾架构

资金流向采用 **3 路径自动容灾**，确保即使东方财富（East Money）拒绝连接也能正常获取数据：

| 优先级 | 路径 | 数据源 | 特点 |
|---|---|---|---|
| **1** | 同花顺 (Tonghuashun) | `ak.stock_fund_flow_individual()` | 独立数据源，不受东方财富限流，~5186 只全量 |
| 2 | AKShare (东方财富) | `ak.stock_individual_fund_flow_rank()` | 含超大单/大单拆分，但 ~55 页任一页失败即整体挂 |
| 3 | 东方财富直连 API | `push2.eastmoney.com` 分页并行抓取 | 页级重试 + 并发，单页失败不影响其他页 |

路径选择逻辑：上一路径失败自动切换下一路径。每条路径成功后，DataFrame 会标记 `attrs["flow_source"]`（值为 `"tonghuashun"` / `"akshare"` / `"eastmoney"`），该标记会持久化到缓存元数据 `cache/YYYYMMDD/.cache_meta.json` 中。

**同花顺与东方财富的数据差异：**

| 字段 | 东方财富 | 同花顺 | 处理方式 |
|---|---|---|---|
| 主力净流入 | ✅ 元 | ✅ 亿/万（字符串） | 解析单位后转为万元 |
| 超大单净流入 | ✅ | ❌ 无此数据 | 代理为 `净额 * 0.5` |
| 大单净流入 | ✅ | ❌ 无此数据 | 代理为 `净额 * 0.5` |
| 流入率 | ✅ % | ❌ 无此数据 | `净额 / 成交额 * 100` 计算 |
| 成交额 | ❌ | ✅ 亿/万（字符串） | 解析单位后转为万元 |

---

## 4. 配置文件说明

### 4.1 config.json

```json
{
  "pandadata": {
    "base_url": "http://pandadata.pandaaiquant.com",
    "username": "",
    "password": ""
  },
  "scan": {
    "min_market_cap": 30,
    "min_turnover": 5000,
    "min_data_rows": 30,
    "top_n": 20
  },
  "patterns": {
    "ma_golden_cross": true,
    "macd_golden_cross": true,
    "bullish_alignment": true,
    "volume_breakout": true,
    "bollinger_breakout": true,
    "hammer": true,
    "morning_star": true,
    "rsi_oversold": true
  },
  "flow": {
    "main_inflow_rate_min": 0.05,
    "super_large_positive": true
  },
  "llm": {
    "model": "claude-sonnet-4-6",
    "max_tokens": 1536
  },
  "output": {
    "dir": "output"
  },
  "detector_weights": {
    "ma_golden_cross": 2,
    "macd_golden_cross": 1,
    "bullish_alignment": 1,
    "volume_breakout": 3,
    "bollinger_breakout": 3,
    "hammer": 1,
    "morning_star": 3,
    "rsi_oversold": 1
  },
  "backtest": {
    "last_run": "2026-06-30",
    "n_stocks": 1000,
    "n_records": 233894,
    "method": "Spearman rank IC (5d forward return)",
    "note": "Weights calibrated against real cache data. Re-run quarterly."
  }
}
```

### 4.2 字段说明

| 路径 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `scan.min_market_cap` | float | `30` | 最低市值（亿元） |
| `scan.min_turnover` | float | `5000` | 最低成交额（万元） |
| `scan.min_data_rows` | int | `30` | 最低K线数据行数 |
| `scan.top_n` | int | `20` | 报告中入选个股数量上限 |
| `patterns.*` | bool | `true` | 各形态检测器开关，设为 `false` 跳过 |
| `flow.main_inflow_rate_min` | float | `0.05` | 主力净流入率阈值（小数，0.05=5%） |
| `flow.super_large_positive` | bool | `true` | 是否要求超大单净流入 > 0 |
| `llm.model` | string | — | 模型名称，会被环境变量 `ANTHROPIC_MODEL` 覆盖 |
| `llm.max_tokens` | int | `1536` | LLM 单次响应最大 token 数（含 DeepSeek 思考 token） |
| `output.dir` | string | `output` | 报告输出根目录（日期子文件夹自动创建） |
| `detector_weights.*` | int | — | 各形态检测器权重，由回测脚本校准后写入 |
| `backtest.last_run` | string | — | 最近一次回测校准日期 |
| `backtest.n_stocks` | int | — | 回测样本股票数 |
| `backtest.n_records` | int | — | 回测信号记录总数 |
| `backtest.method` | string | — | 回测方法说明 |
| `backtest.note` | string | — | 回测备注 |

### 4.3 环境变量

| 变量 | 用途 | 优先级 |
|---|---|---|
| `DEFAULT_USERNAME` | Pandadata 用户名 | 高于 config.json |
| `DEFAULT_PASSWORD` | Pandadata 密码 | 高于 config.json |
| `JAVA_SERVICE_BASE_URL` | Pandadata API 地址 | 高于 config.json |
| `ANTHROPIC_BASE_URL` | LLM API 地址（DeepSeek 需设） | 高于 config.json |
| `ANTHROPIC_AUTH_TOKEN` | LLM API 密钥 | 最高优先 |
| `ANTHROPIC_API_KEY` | LLM API 密钥（备选） | 次优先级 |
| `ANTHROPIC_MODEL` | LLM 模型名称 | 高于 config.json |

---

## 5. 数据流

```
                    ┌─────────────────────┐
                    │  config.json / env  │
                    └─────────┬───────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  K线数据       │   │  资金流向      │   │  股票信息      │
│  Pandadata    │   │  同花顺(主)    │   │  Pandadata    │
│  get_stock_   │   │  → AKShare    │   │  get_stock_   │
│  daily        │   │  → 东方财富直连 │   │  detail       │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘
        │                   │                   │
        ▼                   │                   │
┌───────────────┐           │                   │
│  8 形态检测器  │           │                   │
│  pattern_     │           │                   │
│  detector.py  │           │                   │
└───────┬───────┘           │                   │
        │                   ▼                   │
        │           ┌───────────────┐           │
        │           │  资金过滤器    │           │
        │           │  FlowFilter   │           │
        │           │  流入率>5%    │           │
        │           │  成交额>5000万 │           │
        │           │  超大单>0     │           │
        │           └───────┬───────┘           │
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │ 交叉验证
                            ▼
                    ┌───────────────┐
                    │  评分 + 排名   │
                    │  scorer.py    │
                    │  Z-score 行业  │
                    │  中性化        │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  LLM 逐股分析  │
                    │  llm/analyst  │
                    │  DeepSeek v4  │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │  报告输出      │
                    │  .md + .json  │
                    │  + 数据溯源表  │
                    │  + 验证器      │
                    └───────────────┘
```

---

## 6. 评分公式

```
总分 = pattern_score + flow_score + quality_bonus
```

### 6.1 pattern_score（形态得分，行业中性化后）

| 形态 | 权重 | 触发条件 |
|---|---|---|
| 放量突破 | 3 | 收盘 20 日新高 + 量 > 1.5×5 日均量 |
| 布林带突破 | 3 | 带宽扩张 + 收盘突破上轨 |
| 启明星 | 3 | 阴→小实体→阳 三日反转 |
| 均线金叉 | 2 | MA5 上穿 MA20 |
| RSI超卖反弹 | 1 | RSI(14) < 30 + 当日收阳 |
| MACD金叉 | 1 | DIF 上穿 DEA |
| 多头排列 | 1 | MA5 > MA10 > MA20 > MA60 |
| 锤子线 | 1 | 下影线 ≥ 2×实体 + 下跌趋势中 |

行业中性化：`neutralized = max(0, (raw − μ_industry) / σ_industry)`，剔除行业偏差后地板为 0。

### 6.2 flow_score（资金得分）

```
flow_score = min(main_inflow_rate / 0.05, 3.0)
```

- `main_inflow_rate` 为小数（如 0.11 = 11%）
- 11% → 0.11 / 0.05 = 2.2 分
- 封顶 3.0 分（即流入率 ≥ 15% 后不再加分）

### 6.3 quality_bonus（质量加分）

| 条件 | 加分 |
|---|---|
| 市值 ≥ 50 亿 | +1 |
| 成交额 ≥ 20000 万（2 亿） | +1 |

---

## 7. 形态检测器详情

### 7.1 检测器配置

在 `config.json` 的 `patterns` 中单独开关：

```json
"patterns": {
  "ma_golden_cross": true,       // 均线金叉
  "macd_golden_cross": true,     // MACD金叉
  "bullish_alignment": true,     // 多头排列
  "volume_breakout": true,       // 放量突破
  "bollinger_breakout": true,    // 布林带突破
  "hammer": false,               // 锤子线（可关闭）
  "morning_star": true,          // 启明星
  "rsi_oversold": true           // RSI超卖反弹
}
```

### 7.2 数据窗口要求

- K 线数据最少 **30 个交易日**
- 正式模式拉取 **120 天**数据以确保指标计算精度
- MA60 计算需要至少 60 根 K 线
- RSI(14) 需要至少 15 根 K 线

---

## 8. 缓存机制

系统使用 Parquet 格式按日期缓存数据（`cache/YYYYMMDD/`），避免重复拉取全市场数据：

```
cache/
└── 20260630/
    ├── kline.parquet            # K线数据 (820K+ rows)
    ├── flow.parquet             # 资金流向数据
    ├── info.parquet             # 股票信息数据
    └── .cache_meta.json         # 缓存元数据（含 flow_source）
```

- 同一天多次运行自动复用缓存
- 缓存为空时自动重新拉取（如上次资金流向获取失败）
- 缓存超过 30 天自动清理

---

## 9. 输出文件

### 9.1 目录结构

```
output/
└── 2026-06-30/
    ├── daily_screener_20260630.md    # Markdown 日报
    └── daily_screener_20260630.json  # JSON 结构化数据
```

### 9.2 Markdown 报告结构

| 章节 | 内容 |
|---|---|
| 标题 | 日期 + 扫描统计 |
| 扫描配置 | 参数：市值、成交额、检测器列表、流入率阈值 |
| 今日精选 | Top N 个股：评分、LLM 分析、触发形态、资金信号 |
| 形态触发统计 | 每种形态触发次数和占比 |
| 行业分布 | 入选个股的行业分布 |
| **数据来源** | 各类数据的实际来源和真实性状态（含 ✅/❌ 标注） |
| 数据说明 | 数据截止时间、评分公式、缺失或降级说明 |
| 免责声明 | 风险提示 |

### 9.3 JSON 结构

```json
{
  "trade_date": "20260630",
  "scan_time": "2026-06-30T16:33:50",
  "total_stocks": 5186,
  "selected_count": 10,
  "data_provenance": {
    "kline": {"source": "Pandadata get_stock_daily", "is_real": true},
    "flow": {"source": "同花顺 stock_fund_flow_individual (10jqka)", "is_real": true},
    "stock_info": {"source": "Pandadata get_stock_detail / get_trade_list", "is_real": true},
    "llm": {"source": "Claude / DeepSeek API（真实大模型分析）", "is_real": true},
    "all_real": true
  },
  "stocks": [
    {
      "code": "300503.SZ",
      "name": "昊志机电",
      "industry": "工业",
      "market_cap": 284.0,
      "close": 92.30,
      "pct_change": 19.99,
      "triggered_patterns": ["放量突破", "布林带突破", "启明星"],
      "main_inflow": 65700,
      "inflow_rate": 18.37,
      "super_large_inflow": 32850,
      "turnover": 357600,
      "vol_ratio": 2.06,
      "score": 9.0,
      "pattern_score": 5.0,
      "pattern_score_raw": 11,
      "flow_score": 3.0,
      "quality_bonus": 1,
      "industry_adj": 3.5,
      "industry_std": 2.1,
      "rank": 1,
      "llm_analysis": "昊志机电今日触发多重技术信号共振..."
    }
  ],
  "pattern_stats": {
    "多头排列": 246,
    "RSI超卖反弹": 199,
    "布林带突破": 115,
    "MACD金叉": 87,
    "启明星": 84,
    "放量突破": 61,
    "均线金叉": 22,
    "锤子线": 3
  },
  "industry_distribution": {
    "信息技术": 292,
    "工业": 152,
    "原材料": 63
  }
}
```

### 9.4 字段单位

| 字段 | 单位 | 示例 |
|---|---|---|
| `market_cap` | 亿元 | `284.0` = 284 亿 |
| `main_inflow` | 万元 | `65700` = 6.57 亿 |
| `super_large_inflow` | 万元 | `32850` = 3.29 亿 |
| `turnover` | 万元 | `357600` = 35.76 亿 |
| `inflow_rate` | % | `18.37` = 18.37% |
| `pct_change` | % | `19.99` = +19.99% |
| `score` | 分 | `9.0` |

---

## 10. 验证

### 10.1 自动验证

`run.py` 报告保存后自动调用验证器。生产模式默认严格模式。

### 10.2 手动验证

```bash
# 标准模式
python scripts/validate_screener.py output/2026-06-30/daily_screener_20260630.md output/2026-06-30/daily_screener_20260630.json

# 严格模式（生产推荐）
python scripts/validate_screener.py output/2026-06-30/daily_screener_20260630.md output/2026-06-30/daily_screener_20260630.json --strict
```

### 10.3 检查项

| 检查 | 模式 |
|---|---|
| MD 必需章节（标题、配置、精选、统计、行业、数据来源、说明） | 全部 |
| JSON 必需顶层字段（含 `data_provenance`） | 全部 |
| JSON 每只股票必需字段（含评分子项） | 全部 |
| 评分一致性：`score ≈ pattern_score + flow_score + quality_bonus` | 全部 |
| 免责声明 | 全部 |
| 主力流入率 > 5% | 仅 `--strict` |
| 成交额 > 5000 万 | 仅 `--strict` |
| 超大单净流入 > 0 | 仅 `--strict` |

---

## 11. LLM 分析

### 11.1 配置

| 配置项 | 来源 |
|---|---|
| API 地址 | `ANTHROPIC_BASE_URL` 环境变量 |
| API 密钥 | `ANTHROPIC_AUTH_TOKEN` 或 `ANTHROPIC_API_KEY` |
| 模型 | `ANTHROPIC_MODEL` 环境变量 或 config.json `llm.model` |
| Token 上限 | config.json `llm.max_tokens`（默认 1536） |

### 11.2 降级策略

- DeepSeek 思考 token 消耗部分 `max_tokens` 配额，约 **15%** 的个股可能返回空文本
- 空响应自动触发 fallback 模板，生成一句话摘要
- 单只股票 LLM 失败 **不会阻塞**其余股票的分析

### 11.3 API 调用

- 逐只串行调用，间隔 **0.5 秒**
- 每只股票消耗 1 次 API 请求
- Top 10 完整分析约耗时 **60–80 秒**（含网络延迟）

---

## 12. 测试

```bash
# 运行全部 154 个测试
python -m pytest tests/ -v

# 按模块运行
python -m pytest tests/test_pattern_detector.py -v   # 45 个形态检测器测试
python -m pytest tests/test_integration.py -v          # 32 个流水线/集成测试
python -m pytest tests/test_analyst.py -v              # 24 个 LLM 分析师测试
python -m pytest tests/test_api_integration.py -v      # 25 个 API 集成测试（含 3 路径容灾）
python -m pytest tests/test_reporter.py -v             # 28 个报告生成测试
```

---

## 13. 常见问题

### Q1: 资金流向数据为空怎么办？

系统自动使用 **3 路径容灾**：同花顺 → AKShare/东方财富 → 东方财富直连 API。只要同花顺服务正常，就能获取数据。如果三条路径全部失败，报告会在"数据说明"中标注，且资金过滤条件放宽（flow_score = 0）。

### Q2: LLM 分析为空/乱码？

检查：
1. `ANTHROPIC_AUTH_TOKEN` 是否正确
2. `ANTHROPIC_BASE_URL` 是否设为 `https://api.deepseek.com/anthropic`
3. `config.json` 中 `llm.max_tokens` 是否 ≥ 1536
4. 查看终端日志中是否有 `WARNING: LLM returned empty text`

### Q3: 报告只有寥寥几只股票？

正式模式扫描全市场 5000+ 只股票。如果结果很少，可能是当日市场行情偏弱、资金流入普遍低迷。可尝试放宽资金过滤阈值。

### Q4: Pandadata 登录失败？

错误 `200006 用户未注册` 表示该手机号未在 Pandadata API 服务注册。需要：
1. 访问 `http://pandadata.pandaaiquant.com` 注册
2. 确认手机号前有 `86` 前缀（如 `8613800138000`）

### Q5: 如何关闭某些形态检测器？

在 `config.json` 中将对应 `patterns` 字段设为 `false`。例如只保留趋势类形态：
```json
"patterns": {
  "ma_golden_cross": true,
  "macd_golden_cross": true,
  "bullish_alignment": true,
  "volume_breakout": true,
  "bollinger_breakout": true,
  "hammer": false,
  "morning_star": false,
  "rsi_oversold": false
}
```

### Q6: 如何修改资金过滤阈值？

在 `config.json` 中修改：
```json
"flow": {
  "main_inflow_rate_min": 0.03,   // 改为 3%
  "super_large_positive": false    // 不要求超大单为正
},
"scan": {
  "min_turnover": 10000            // 改为 1 亿
}
```

### Q7: 同花顺和东方财富的资金数据有什么区别？

同花顺提供的是总流入/流出/净额，没有超大单/大单/中单/小单的拆分。系统将超大单和大单各代理为净额的一半。流入率通过 `净额 / 成交额 * 100` 计算。详见第 3 节的对比表。

---

## 14. 目录结构

```
skill-post-market-screener/
├── SKILL.md                    # Agent 工作流入口（AI 阅读）
├── README.md                   # 项目介绍（人类阅读）
├── OPERATION_MANUAL.md         # 本手册
├── config.json                 # 运行配置
├── pyproject.toml              # Python 项目配置
├── requirements.txt            # Python 依赖
├── run.py                      # 主入口（仅真实数据模式）
├── mcp_server.py               # MCP Server
├── core/
│   ├── data_fetcher.py         # 数据获取：Pandadata K线+股票信息（含重试+退避）
│   ├── flow_fetcher.py         # 资金流向：同花顺→AKShare→东方财富直连 3路径容灾
│   ├── pattern_detector.py     # 8 个形态检测器（连续强度信号）
│   ├── flow_filter.py          # 资金流向过滤器
│   ├── scorer.py               # 评分 + 排名（Z-score 行业中性化）
│   ├── reporter.py             # Markdown + JSON 报告生成（含数据溯源表）
│   ├── pipeline.py             # 端到端流水线编排器
│   ├── cache.py                # 日期分组 Parquet 缓存（含 .cache_meta.json）
│   └── mock_data.py            # Mock 数据生成器（内部测试用）
├── llm/
│   └── analyst.py              # LLM 分析（DeepSeek/Claude，并发调用）
├── tests/
│   ├── test_pattern_detector.py   # 45 个形态检测器测试
│   ├── test_integration.py        # 32 个流水线/集成测试
│   ├── test_analyst.py            # 24 个 LLM 分析师测试
│   ├── test_api_integration.py    # 25 个 API 集成测试（含 3 路径容灾）
│   └── test_reporter.py           # 28 个报告生成测试
├── scripts/
│   ├── validate_screener.py    # 输出完整性校验器
│   ├── daily_screener.bat      # Windows Task Scheduler 定时任务入口
│   ├── benchmark.py            # 性能基准测试
│   └── analyze_weights.py      # IC 分析与权重优化
├── references/
│   ├── pandadata-map.md        # Pandadata 接口路由表
│   ├── pattern-formulas.md     # 形态公式详解
│   └── report-template.md      # 报告模板 + LLM Prompt
├── agents/                     # 多平台适配器
│   ├── openai.yaml
│   ├── cursor-rule.mdc
│   └── portable-loader.md
├── cache/                      # 数据缓存（按日期分组）
│   └── YYYYMMDD/
│       ├── kline.parquet
│       ├── flow.parquet
│       ├── info.parquet
│       └── .cache_meta.json
└── output/                     # 报告输出（按日期分组）
    ├── 2026-06-29/
    │   ├── daily_screener_20260629.md
    │   └── daily_screener_20260629.json
    └── 2026-06-30/
        ├── daily_screener_20260630.md
        └── daily_screener_20260630.json
```

---

## 15. 免责声明

本 Skill 输出仅供研究参考，**不构成任何投资建议**。所有数据来自公开接口，不保证实时性、准确性和完整性。投资者应独立判断并承担交易风险。股市有风险，投资需谨慎。
