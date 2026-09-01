---
name: post-market-screener
description: Daily A-share end-of-day quantitative screener that finds stocks with both
  technical-pattern breakouts and institutional capital inflows. Covers 8 pattern detectors
  (MA/MACD cross, bullish alignment, volume breakout, Bollinger breakout, hammer, morning
  star, RSI oversold), main-capital flow filtering (Tonghuashun primary, East Money fallback),
  dual-factor cross-validation, z-score industry neutralization, and per-stock LLM analysis
  (Claude/DeepSeek). Use when the user asks for 收盘扫描, 选股扫描, 形态选股,
  post-market screening, 资金+形态筛选, or to set up automated daily stock screening.
version: 2.0.0
category: quant-skills
triggers:
  - 收盘扫描
  - 选股扫描
  - 形态选股
  - post-market screening
  - 资金+形态筛选
  - 每日选股
data_sources:
  - pandadata (K-line, stock info, trading calendar)
  - tonghuashun/10jqka (fund flow, primary)
  - eastmoney via akshare (fund flow, fallback)
  - eastmoney direct push2 API (fund flow, last-resort)
  - deepseek/claude (LLM per-stock analysis)
output_formats:
  - markdown日报
  - json结构化数据
schedule: 每日收盘后 15:45 Asia/Shanghai（盘后固定价格交易 15:05-15:30 结束后执行）
license: GPLv3
metadata:
  organization: QuantSkills
  organization_url: https://github.com/quantskills
  repository: skill-post-market-screener
  repository_url: https://github.com/quantskills/skill-post-market-screener
  project_type: skill
  collection: post-market-screener
  creator: Tao
quantSkills:
  project_type: skill
  category: analysis
  tags:
  - a-share
  - technical-patterns
  - capital-flow
  - screening
  - pandadata
  - tonghuashun
  platforms:
  - claude-code
  - codex
  - hermes
  - openclaw
  - cursor
  status: dev
  validation_level: runnable
  maintainer_type: community
  summary_zh: >-
    每日收盘后自动扫描全市场，8 个技术形态检测器（连续强度信号）+ 同花顺主力资金流入做双因子交叉验证，Z-score 行业中性化排名，Top N 个股 DeepSeek LLM 分析，输出 Markdown 日报 + JSON 数据。资金流向 3 路径容灾：同花顺→东方财富 AKShare→东方财富直连。
  summary_en: >-
    Daily A-share post-market screener: 8 technical pattern detectors (continuous
    float strength) x Tonghuashun capital inflow filter = dual-factor cross-validated
    stock picks with DeepSeek LLM analysis and z-score industry neutralization.
    Fund flow uses 3-path fallback: Tonghuashun → East Money (AKShare) → East Money direct.
  license: GPLv3
  version: 2.0.0
---

# Post-Market Screener

After each A-share trading day, scan the entire market with dual-factor (technical pattern + capital flow) cross-validation, then generate a ranked daily screening report with per-stock LLM analysis.

## Workflow

1. **Determine the target date.** If the user does not provide one, use the latest completed A-share trading day. Check `get_last_trade_date` and `get_trade_cal`; if the target date is closed, return a short "今日休市" note instead of running the scan.
2. **Collect data** in this order:
   - Trading calendar and stock universe: `get_last_trade_date`, `get_trade_cal`, `get_trade_list`.
   - Daily K-line (120 days for pattern calculation window): `get_stock_daily`.
   - Fund flow per stock: **同花顺 (Tonghuashun/10jqka)** `stock_fund_flow_individual` as primary source. Falls back to AKShare `stock_individual_fund_flow_rank` (东方财富), then direct East Money push2 API on failure. The flow source is tracked via `df.attrs["flow_source"]` and persisted in cache metadata.
   - Stock basic info (name, industry): `get_stock_detail`; market cap computed from `get_share_float` × close price. **排除 ST / \*ST / PT / 退市整理期股票**（通过 `list_status` 字段检测，2026年7月 ST 涨跌幅放宽至 ±10% 后波动加剧，形态信号可靠性下降，排除策略暂不调整）。
3. **Run 8 pattern detectors** against each stock's K-line data (formulas in `references/pattern-formulas.md`). Each detector returns a continuous float strength value (> 0 = triggered, ≤ 0 = not triggered). The strength magnitude is used for backtest IC calculation; scoring uses binary weights.
   - MA Golden Cross: MA5 crosses above MA20
   - MACD Golden Cross: DIF crosses above DEA
   - Bullish Alignment: MA5 > MA10 > MA20 > MA60
   - Volume Breakout: close at 20-day high + volume > 1.5x 5-day avg volume
   - Bollinger Breakout: bandwidth expansion + price breaks upper band
   - Hammer: lower shadow >= 2x body + in downtrend
   - Morning Star: bear candle → small body → bull candle (3-day reversal)
   - RSI Oversold Rebound: RSI(14) < 30 + today closes bullish
4. **Filter by capital flow:**
   - Main capital net inflow rate > 5%
   - Turnover > 50M CNY (exclude illiquid stocks)
   - Super-large order net inflow > 0
   Note: Tonghuashun does not provide order-size breakdown (super_large/large/medium/small). Super-large and large net inflows are proxied as `main_net_inflow / 2` each.
5. **Cross-validate and rank.** A stock must pass at least one pattern AND the flow filter. Compute `stock_score = neutralized_pattern_score + flow_score + quality_bonus`, then take the Top N.
6. **Generate per-stock LLM analysis** using `references/report-template.md` for the analysis prompt. Each stock gets detailed Chinese analysis (technical breakdown + fund verification + sector context + risk notes).
7. **Render the report** with data provenance table showing the actual source for each data category. Save to `output/YYYY-MM-DD/daily_screener_YYYYMMDD.md` and `output/YYYY-MM-DD/daily_screener_YYYYMMDD.json` unless the user gives another path.
8. **Run `scripts/validate_screener.py <md-path> <json-path>`** after writing the output. Production runs use `--strict` mode. The validator checks 17 rules across 5 categories (结构完整性 / 数据溯源 / 评分可审计性 / 内容合规 / 数据质量). See `references/validate-rules.md` for the complete rule checklist and exit codes. Fix any ERROR-level violations before presenting the result.
9. **Periodic backtest & weight calibration.** After accumulating ≥60 trading days of fresh K-line data in cache, run `scripts/analyze_weights.py --data cache/ --json` to compute per-detector Spearman rank IC (1d/3d/5d/10d/20d forward returns). Save the resulting weights to `config.json` under `detector_weights`. The pipeline automatically loads calibrated weights via `apply_weights_from_config()` at startup. Re-run quarterly to keep weights current.

> **回测方法说明（防止前视偏差）：** 所有前向收益从 **T+1 开盘价** 起算（而非 T 日收盘），排除当日跳空和不可交易时段。例如 5d forward return = (T+6 收盘 − T+1 开盘) / T+1 开盘。这确保了 IC 计算结果反映的是**实际可实现的**交易收益，而非包含 T 日收盘已锁定的假收益。

> **生存偏差提示：** 回测使用的股票 universe 来自回测日期的 `get_trade_list`，这意味着已退市股票不会出现在历史扫描中。在长周期回测（>1年）中，生存偏差可能导致 IC 被系统性高估约 0.02–0.05。建议在对比新规前后表现时，使用固定 universe（回测起点当日全 A 股列表）来消除此偏差。

> **2026-07-06 新规后校准：** 主板 ST 涨跌幅由 ±5%→±10%、盘后固定价格交易扩容两项变更后，建议在 2026年10月（积累约 60 个交易日后）运行一次专项回测，对比新规前后的 IC 和权重变化。使用 `scripts/analyze_weights.py --since 2026-07-06 --compare-pre-regulation` 输出前后对比报告。

## Pandadata Reference

Read `references/pandadata-map.md` when planning calls, selecting fields, or deciding how to degrade if a data interface is unavailable. The map documents which Pandadata endpoint serves each data requirement.

## Scoring Formula

```
pattern_score = sum(triggered_pattern_weights)   # weights from config.json detector_weights or DETECTOR_REGISTRY defaults
flow_score    = min(main_inflow_rate_decimal / 0.05, 3)  # 1 point per 5% inflow rate, capped at 3
quality_bonus:
  +1  if market_cap >= 50B CNY
  +1  if turnover >= 200M CNY

Stage 1 — Industry neutralization (z-score within each industry):
  z_ind = (raw_pattern_score − μ_industry) / σ_industry
  z_ind = max(0, z_ind)   # floor at 0

Stage 2 — Market-cap neutralization (z-score within each market-cap quintile) [v2.1]:
  z_mcap = (z_ind − μ_mcap_quintile) / σ_mcap_quintile
  z_mcap = max(0, z_mcap)   # floor at 0

Blend:
  neutralized = z_ind × 0.6 + z_mcap × 0.4

Final score = neutralized_pattern_score + flow_score + quality_bonus
```

> **市场中性化说明 (v2.1)：** A 股小市值因子异常显著——小盘股在技术形态检测器中系统性触发率更高（波动大、均线交叉频繁）。单纯行业中性化无法消除这个偏差。v2.1 引入市值五分位 Z-score 作为第二层中性化，60/40 混合权重保留更多行业信号同时修正小盘股优势。

## Portfolio Optimization [v2.1]

After scoring and ranking, the pipeline computes three portfolio allocation schemes for the Top-N stocks:

| Scheme | Method | Best For |
|--------|--------|----------|
| **等权 (Equal Weight)** | 1/N allocation | Benchmark, simplest |
| **最小方差 (Min Variance)** | Minimize w'Σw via quadratic programming, long-only | Conservative, low-volatility |
| **风险平价 (Risk Parity)** | Equal risk contribution via iterative scaling | Balanced risk exposure |

The covariance matrix is estimated from 60-day daily log-returns. Portfolio weights are included in the JSON output and rendered as a comparison table in the Markdown report.

## Report Rules

- Write in Chinese unless the user requests another language.
- Use absolute dates such as `2026-06-30`; avoid ambiguous "today" in the final report body.
- Each selected stock must show: triggered patterns, capital flow data, and the LLM-generated analysis.
- Use cautious language: "值得关注", "可跟踪", "信号偏多"; never say "推荐买入", "目标价", "必涨".
- When a data call fails, keep the report useful by skipping that stock (not the whole scan) and adding a missing-data count under "数据说明".
- Include scoring breakdown in the JSON output for auditability.
- Always include a **数据来源 (data provenance)** table showing the actual source and authenticity status for each data category.

## Automation

When the user asks for automated daily screening, create an after-close task for trading days only, preferably after `15:45 Asia/Shanghai` (15 min after 盘后固定价格交易结束). A `scripts/daily_screener.bat` entry point for Windows Task Scheduler is provided. Make the task idempotent: if `output/YYYY-MM-DD/daily_screener_YYYYMMDD.md` already exists, regenerate and overwrite it.

> **2026-07-06 新规提示：** 盘后固定价格交易已扩容至全部 A 股（15:05-15:30），收盘价和数据在 15:30 后才真正锁定。Pandadata `get_stock_daily` 的 K 线数据是否包含盘后固定价格成交**需要在首次运行时验证**（检查 `close` 字段与行情软件收盘价是否一致）。如有差异，在报告「数据说明」节中标注。

## Directory Structure

```
skill-post-market-screener/
├── SKILL.md                         # Agent workflow entry point
├── README.md                        # Human documentation
├── OPERATION_MANUAL.md              # Detailed operation manual
├── LICENSE                          # GPLv3
├── config.json                      # Runtime config + backtest-calibrated weights
├── pyproject.toml                   # Python project metadata + linting config
├── .env.example                     # Environment variable template
├── requirements.txt                 # Python dependencies
├── run.py                           # CLI entry point (real data only, no --mock/--dry-run)
├── mcp_server.py                    # MCP server
├── core/
│   ├── data_fetcher.py              # Pandadata data acquisition (with retry + backoff)
│   ├── flow_fetcher.py              # Fund flow: 同花顺 → AKShare → East Money direct (3-path)
│   ├── pattern_detector.py          # 8 technical pattern detectors (float strength)
│   ├── flow_filter.py               # Capital flow filter + inflow rate normalization
│   ├── scorer.py                    # Scoring and ranking (z-score industry + mcap neutralization)
│   ├── reporter.py                  # Markdown + JSON report generator (data provenance + portfolio)
│   ├── pipeline.py                  # End-to-end pipeline orchestrator
│   ├── portfolio.py                 # Portfolio optimization (equal-weight, min-variance, risk-parity) [v2.1]
│   ├── cache.py                     # Date-keyed Parquet cache (with meta.json)
│   └── mock_data.py                 # Mock data generator (internal testing only)
├── llm/
│   └── analyst.py                   # LLM analysis (Claude/DeepSeek, concurrent API calls)
├── tests/
│   ├── test_pattern_detector.py     # 45 pattern detector tests
│   ├── test_integration.py          # 32 pipeline/integration tests
│   ├── test_analyst.py              # 24 LLM analyst tests
│   ├── test_api_integration.py      # 25 mocked API integration tests (3-path fallback)
│   └── test_reporter.py             # 28 report generation tests
├── references/
│   ├── pandadata-map.md             # Data routing reference
│   ├── pattern-formulas.md          # Detector formulas and weights
│   ├── report-template.md           # Report template + LLM prompt
│   └── validate-rules.md            # Output integrity validator rule checklist
├── scripts/
│   ├── validate_screener.py         # Output integrity validator
│   ├── daily_screener.bat           # Windows Task Scheduler entry point
│   ├── benchmark.py                 # Performance benchmark
│   └── analyze_weights.py           # IC analysis and weight optimization
├── agents/
│   ├── openai.yaml                  # OpenAI/Codex adapter
│   ├── cursor-rule.mdc              # Cursor IDE adapter
│   └── portable-loader.md           # Generic loader for any agent
└── output/                          # Reports grouped by date
    └── YYYY-MM-DD/
        ├── daily_screener_YYYYMMDD.md
        └── daily_screener_YYYYMMDD.json
```

## Core Constraints

| Constraint | Description |
|---|---|
| Dual-factor must hold | Stock passes only if pattern AND flow both trigger |
| ST stock exclusion | **ST / \*ST / PT / 退市整理期股票直接排除**。2026年7月6日起 ST 涨跌幅由 ±5%→±10%，波动空间翻倍，形态检测器假阳性率上升。待积累 ≥60 个交易日新规后数据后运行权重回测，再评估是否重新纳入 |
| ST liquidity threshold | 若未来重新纳入 ST 股，其 `min_turnover` 阈值需加倍至 100M CNY（涨跌幅放宽后流动性门槛相应提高） |
| Liquidity threshold | Turnover < 50M CNY → auto-exclude（非ST普通股） |
| No stock recommendation | Use "值得关注" / "可跟踪", forbid "买入" / "目标价" |
| Trading-day aware | Skip holidays, check trade calendar |
| Data fault tolerance | Skip individual failing stocks, do not halt the full scan |
| Audit trail | JSON output must include scoring breakdown per stock (raw + neutralized) |
| Data provenance | Report must label each data category's actual source and authenticity |
| Post-market data timing | **2026-07-06起：** 盘后固定价格交易（15:05-15:30）扩容至全部 A 股，触发时间调整为 15:45。需验证 Pandadata K 线数据是否包含盘后成交 |
| **Survivorship bias** | 股票 universe 来自 `get_trade_list`（当日可交易），已退市股票不在历史回测中。长周期回测的 IC 可能被高估 0.02–0.05。使用固定起点 universe 可消除此偏差 |
| **Data freshness** | 数据拉取后校验目标日期数据时间戳：K 线的最大 `trade_date` ≠ 目标日期 → 数据尚未就绪，等待 60s 后重试（最多 3 次）。全部重试失败 → 报告标注「数据延迟」+ 实际最新数据日期 |
| **Runtime SLA** | 全市场扫描（~5000 只）：预期 8–15 分钟（4 线程 + Parquet 缓存命中）。>30 分钟 → WARNING 记录 API 延迟。>60 分钟 → 超时退出，输出已完成的部分结果 |

## Disclaimer

This skill's output is for research reference only. It does not constitute any investment advice. Investors should make independent judgments and bear trading risks.

## License

GPLv3
