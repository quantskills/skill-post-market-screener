# Post-Market Screener Skill

> Daily A-share end-of-day quantitative screener: 8 technical pattern detectors × capital inflow filter = dual-factor cross-validated stock picks with DeepSeek/Claude LLM analysis.

<p align="center">
  <img alt="pattern detectors" src="https://img.shields.io/badge/pattern_detectors-8-brightgreen">
  <img alt="filter layers" src="https://img.shields.io/badge/filter_layers-3-blue">
  <img alt="output formats" src="https://img.shields.io/badge/outputs-Markdown_+_JSON-orange">
  <img alt="automation" src="https://img.shields.io/badge/automation-after--close_cron-9cf">
  <img alt="requires" src="https://img.shields.io/badge/requires-pandadata--api-7c3aed">
  <img alt="tests" src="https://img.shields.io/badge/tests-154-green">
  <img alt="mcp" src="https://img.shields.io/badge/MCP-3_tools-purple">
  <img alt="license" src="https://img.shields.io/badge/license-GPLv3-blue">
</p>

---

## What is this?

`post-market-screener` is an **Agent Skill**: after each A-share trading day, it scans the entire market using Pandadata + Tonghuashun (同花顺) data. Eight technical pattern detectors and institutional capital inflow filtering are combined for **dual-factor cross-validation**, producing a ranked daily screening report with per-stock LLM analysis.

Differentiation from other skills:

| Existing Skills | Why This is Different |
|---|---|
| Daily Market Review | It provides market-level summaries; this Skill performs **stock-level pattern × capital dual screening** |
| A-Share Stock Picker | Natural language interactive queries; this Skill is **automated daily scanning with ranked output** |
| Skill Xingtai Catcher | Based on K-line screenshots/hand-drawn charts for pattern matching; this Skill uses **quantitative indicator-based rule detection** |
| Dragon-Tiger Board Tracking | Tracks only dragon-tiger board seats; this Skill does **market-wide main capital net inflow filtering** |

Core differentiation: **Technical Patterns ∩ Capital Flow = Dual-Factor Cross-Validation**.

---

## Scan Pipeline

```
Trigger (manual or cron 15:45) → Trading Day Check
  → Data Acquisition (K-line + Fund Flow + Stock Info)
  → 8 Pattern Detectors + Capital Flow Filter
  → Dual-Factor Cross-Validation (Patterns ∩ Flow = Selected)
  → Scoring & Ranking (pattern_score + flow_score + quality_bonus)
  → LLM Per-Stock Analysis (DeepSeek/Claude API)
  → Render Report
  → Validate Output (validate_screener.py --strict)
  → Save to output/YYYY-MM-DD/
```

---

## Technical Patterns × Detectors

| # | Detector | Logic | Weight |
|---|---|---|---|
| 1 | MA Golden Cross | MA5 crosses above MA20 | 2 |
| 2 | MACD Golden Cross | DIF crosses above DEA | 1 |
| 3 | Bullish Alignment | MA5 > MA10 > MA20 > MA60 | 1 |
| 4 | Volume Breakout | Close at 20-day high + volume > 1.5×5-day avg | 3 |
| 5 | Bollinger Breakout | Bandwidth expansion + price breaks upper band | 3 |
| 6 | Hammer | Lower shadow ≥ 2× body + in downtrend | 1 |
| 7 | Morning Star | Bear → small body → bull (3-day reversal) | 3 |
| 8 | RSI Oversold Rebound | RSI(14) < 30 + today closes bullish | 1 |

> Detailed formulas in `references/pattern-formulas.md`

### Weight Calibration (Backtest-Validated)

Detector weights are periodically calibrated through historical backtesting, not fixed empirical values:

```bash
# Run backtest (requires ≥60 days of cached data)
python scripts/analyze_weights.py --data cache/ --json

# Check rolling window stability
python scripts/analyze_weights.py --data cache/ --rolling
```

The backtest engine computes for each detector:
- **Rank IC** — Spearman rank correlation between signal strength and forward returns (1/3/5/10/20 day)
- **Hit Rate** — proportion of positive returns after signal trigger
- **IC IR** — Information Ratio of IC (bootstrap × 200)

Optimized weights are written to `config.json` `detector_weights` and auto-loaded by the pipeline. Recalibrate quarterly.

> Latest backtest (2026-06-30): 1000 stock sample, 233,894 records. Morning Star IC(5d)=+0.003 best, MACD Golden Cross IC=+0.016 only significantly positive signal.

---

## Capital Flow Filter Criteria

| Condition | Threshold | Notes |
|---|---|---|
| Main capital net inflow rate | > 5% | Main net inflow / turnover |
| Turnover | > 50M CNY | Exclude illiquid stocks |
| Super-large order net inflow | > 0 | Institutional capital direction confirmation |

---

## Quick Start

### Environment Setup

```bash
git clone <repo-url> skill-post-market-screener
cd skill-post-market-screener
pip install -r requirements.txt
cp .env.example .env   # Edit and fill in your credentials
```

Environment variables in `.env`:

| Variable | Description | How to Get |
|---|---|---|
| `DEFAULT_USERNAME` | Pandadata username | `86` + phone number, register at pandadata.pandaaiquant.com |
| `DEFAULT_PASSWORD` | Pandadata password | Same as above |
| `ANTHROPIC_AUTH_TOKEN` | LLM API key | DeepSeek or Claude API |
| `ANTHROPIC_BASE_URL` | LLM API endpoint | DeepSeek: `https://api.deepseek.com/anthropic` |
| `ANTHROPIC_MODEL` | Model name | `deepseek-v4-pro` or `claude-sonnet-4-6` |

### Method A: CLI

```bash
python run.py                  # Scan latest trading day
python run.py --date 20260629  # Scan a specific date
python run.py --no-flow        # Pattern-only scan (skip capital flow filter)
python run.py --top-n 10       # Report top 10 only
```

Or install via pip:

```bash
pip install .
screener                       # Same as python run.py
screener --date 20260629 --top-n 10
```

### Method B: MCP Server (AI Agent)

Start the MCP Server to let Claude Code / Cursor / Codex and other AI agents call it directly:

```bash
python mcp_server.py
# or after pip install:
screener-mcp
```

Add to your agent's MCP configuration (example for Claude Code, edit `~/.claude/mcp.json`):

```json
{
  "mcpServers": {
    "post-market-screener": {
      "command": "python",
      "args": ["mcp_server.py"],
      "cwd": "/path/to/skill-post-market-screener",
      "env": {
        "ANTHROPIC_AUTH_TOKEN": "sk-xxx",
        "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
        "ANTHROPIC_MODEL": "deepseek-v4-pro",
        "DEFAULT_USERNAME": "86xxxxxxxxxxx",
        "DEFAULT_PASSWORD": "xxx"
      }
    }
  }
}
```

A `.claude/mcp.json` template is included in the project.

MCP Tools:

| Tool | Description |
|---|---|
| `run_screener` | Run full dual-factor market scan (params: `date`, `no_flow`, `top_n`) |
| `get_latest_report` | Read the most recent Markdown report |
| `check_trading_day` | Check if a date is an A-share trading day |

### Scheduled Automation

Windows Task Scheduler (every trading day at 15:37):

```cmd
schtasks /create /tn "PostMarketScreener" /tr "path\to\scripts\daily_screener.bat"
    /sc weekly /d MON,TUE,WED,THU,FRI /st 15:37 /f
```

### Validate Output

```bash
python scripts/validate_screener.py output/2026-06-29/daily_screener_20260629.md output/2026-06-29/daily_screener_20260629.json --strict
```

---

## Data Sources

| Data | Source | Notes |
|---|---|---|
| K-line | Pandadata `get_stock_daily` | Full market ~5186 stocks, 120-day history |
| Fund Flow | **Tonghuashun** `stock_fund_flow_individual` (10jqka) | Main capital net inflow/amount/turnover, independent of East Money |
| Stock Info | Pandadata `get_stock_detail` / `get_trade_list` | Industry, market cap, list status |
| LLM Analysis | DeepSeek API (Anthropic-compatible) | Per-stock technical + capital comprehensive analysis |

Fund flow uses a **3-path fallback architecture**: Tonghuashun (primary) → AKShare/East Money → East Money direct API.

---

## Using with Other AI Agents

The Skill communicates with AI Agents via the **MCP protocol**, compatible with any MCP-compliant agent.

| Agent | Integration | Config File |
|---|---|---|
| **Claude Code** | Add MCP config or place in `.claude/skills/` | `.claude/mcp.json` (template included) |
| **Cursor** | MCP config or `.cursor/skills/` | `agents/cursor-rule.mdc` |
| **Codex / OpenAI** | MCP config | `agents/openai.yaml` |
| **Other MCP Agents** | Standard MCP protocol, configure `mcp_server.py` | — |
| **Non-MCP LLMs** | Inject Portable Loader Prompt | `agents/portable-loader.md` |

All paths use real data (Pandadata + Tonghuashun + LLM API). No mock mode.

---

## Distributing to Others

**Method 1 — GitHub (recommended):**

```bash
# You (author) push:
git remote add origin git@github.com:<your-account>/skill-post-market-screener.git
git push -u origin master

# Others receive:
git clone https://github.com/<your-account>/skill-post-market-screener.git
cd skill-post-market-screener
pip install -r requirements.txt
cp .env.example .env  # Fill in own credentials
python run.py
```

**Method 2 — pip one-liner:**

```bash
pip install git+https://github.com/<your-account>/skill-post-market-screener.git
# Configure .env, then:
screener
```

**Method 3 — Zip:**

```bash
git archive -o post-market-screener.zip HEAD
```

Recipients only need:
- Python 3.10+
- Pandadata credentials (free registration with phone number)
- LLM API Key (DeepSeek or Claude)

---

## Directory Structure

```
skill-post-market-screener/
├── SKILL.md                         # Agent workflow entry point
├── README.md                        # Project introduction (Chinese)
├── README.en.md                     # Project introduction (English)
├── OPERATION_MANUAL.md              # Detailed operation manual
├── LICENSE                          # GPLv3
├── config.json                      # Runtime config + backtest-calibrated weights
├── pyproject.toml                   # Python project metadata + pip entry points
├── requirements.txt                 # Python dependencies
├── .env.example                     # Environment variable template
├── .gitignore                       # Git ignore rules
├── run.py                           # CLI main entry point
├── mcp_server.py                    # MCP Server (3 tools)
├── .claude/
│   └── mcp.json                     # Claude Code MCP config template
├── core/
│   ├── data_fetcher.py              # Data acquisition (Pandadata K-line + stock info)
│   ├── flow_fetcher.py              # Fund flow (Tonghuashun → AKShare → East Money direct)
│   ├── pattern_detector.py          # 8 technical pattern detectors (continuous float strength)
│   ├── flow_filter.py               # Capital flow filter
│   ├── scorer.py                    # Scoring & ranking (z-score industry neutralization)
│   ├── reporter.py                  # Markdown + JSON report generation
│   ├── pipeline.py                  # End-to-end pipeline orchestrator
│   ├── portfolio.py                 # Portfolio optimization (equal/min-var/risk-parity)
│   ├── cache.py                     # Date-keyed Parquet cache
│   └── mock_data.py                 # Mock data generator (internal testing only)
├── llm/
│   └── analyst.py                   # LLM per-stock analysis (DeepSeek/Claude, concurrent)
├── tests/                           # Test suite (154 tests)
├── references/
│   ├── pandadata-map.md             # Data routing reference
│   ├── pattern-formulas.md          # Detector formulas, parameters, weights
│   └── report-template.md           # Daily report template + LLM prompt
├── scripts/
│   ├── validate_screener.py         # Output integrity validator
│   ├── daily_screener.bat           # Windows Task Scheduler entry point
│   ├── benchmark.py                 # Performance benchmark
│   └── analyze_weights.py           # IC analysis & weight optimization
├── agents/
│   ├── openai.yaml                  # OpenAI/Codex adapter
│   ├── cursor-rule.mdc              # Cursor IDE adapter
│   └── portable-loader.md           # Generic loader for any agent
└── output/                          # Reports grouped by date
    └── YYYY-MM-DD/
        ├── daily_screener_YYYYMMDD.md
        └── daily_screener_YYYYMMDD.json
```

---

## Core Constraints

| Constraint | Description |
|---|---|
| Dual-factor must hold | Pattern or flow failing → stock excluded |
| Liquidity threshold | Turnover < 50M CNY → auto-exclude |
| No stock recommendations | Use "值得关注" / "可跟踪", forbid "买入" / "目标价" |
| Trading-day aware | Skip holidays, check trade calendar |
| Data fault tolerance | Skip individual failing stocks, do not halt full scan |
| Audit trail | JSON output must include per-stock scoring breakdown |
| Data provenance | Report must label each data category's actual source |

---

## Disclaimer

This Skill's output is for research reference only. It does **not constitute any investment advice**. Investors should make independent judgments and bear trading risks.

## License

GPLv3
