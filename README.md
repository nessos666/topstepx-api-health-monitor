<p align="center">
  <h1 align="center">TopStepX API Health Monitor</h1>
  <p align="center">
    <strong>Know if your API is healthy before you place a trade. 9 automated checks → single Trust Score 0-100.</strong>
  </p>
  <p align="center">
    <a href="#quick-start">Quick Start</a> · <a href="#the-9-checks">The 9 Checks</a> · <a href="#trust-score">Trust Score</a> · <a href="#alerting">Alerting</a> · <a href="#latency-trending">Latency Trending</a>
  </p>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/python-3.10+-blue?logo=python&logoColor=white" alt="Python 3.10+">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="MIT License">
  <img src="https://img.shields.io/badge/checks-9_automated-orange" alt="9 Checks">
  <img src="https://img.shields.io/badge/trust_score-0--100-red" alt="Trust Score">
  <img src="https://img.shields.io/badge/24x7-systemd_ready-success" alt="systemd">
  <img src="https://img.shields.io/github/stars/nessos666/topstepx-api-health-monitor?style=social" alt="Stars">
</p>

---

> **Built on [api-health-trust-system](https://github.com/nessos666/api-health-trust-system)** — the weighted Trust Score monitoring framework.
> See `examples/topstepx_scanner.py` in that repo for this exact configuration.

## Why?

If you're running algo strategies on TopStepX / ProjectX, you're trusting the API with real money. But APIs go down. Tokens expire silently. Data goes stale. Contracts roll over.

**This tool runs 9 checks every 5 minutes and gives you a single number: a Trust Score from 0 to 100.** If it drops below your threshold, you stop trading. Simple.

Built by an algo trader who got burned by silent API failures. Now open-sourced so you don't have to.

---

## The 9 Checks

Each check maps to a specific failure mode that can silently break your algo:

| # | Check | What it catches | Impact if failed |
|---|-------|----------------|-----------------|
| 1 | **Reachability** | API down? Server error? Timeout? | **Trading impossible** |
| 2 | **Latency** | P95 response time > 2s? Something's wrong. | Orders delayed or dropped |
| 3 | **Data Freshness** | Last bar older than 5 minutes? You're trading blind. | Entries on stale data |
| 4 | **Contract** | Quarterly rollover happened and you missed it? | Trading dead contract |
| 5 | **Token** | JWT expired silently? Auto-detects and warns. | All API calls fail |
| 6 | **canTrade** | Account flagged? Daily loss limit hit? | Orders rejected |
| 7 | **Bar Quality** | NaN values, zero volume, impossible OHLC? | Bad entries from bad data |
| 8 | **Balance** | Drawdown approaching 80%? Time to stop. | Account blowout |
| 9 | **Loop Continuity** | Your live scanner crashed and nobody noticed? | Silent outage |

Each check returns `pass/fail` with a detail message. All 9 feed into the Trust Score.

---

## Trust Score: How It Works

### The Formula

The Trust Score is a **weighted composite** of all 9 checks:

```
trust_score = Σ(weight_i × score_i) / Σ(weight_i) × 100
```

Where `score_i` is 1.0 for pass, 0.0 for fail (some checks have partial scores).

### Weights

| Weight | Checks | Why |
|--------|--------|-----|
| **3x** | Reachability, canTrade | If these fail, you literally cannot trade |
| **2x** | Token, Contract, Loop Continuity | System-level failures that take time to fix |
| **1x** | Latency, Data Freshness, Bar Quality, Balance | Degradation, not outage |

### Thresholds

```
Trust Score    Status        Action
─────────────────────────────────────────────────
80-100         HEALTHY      All systems go. Trade with confidence.
50-79          DEGRADED     Some checks failing. Investigate before trading.
 0-49          CRITICAL     Do NOT trade. Fix issues first.
```

### Real Example

```
Reachability: ✅ HTTP 200 (weight 3)
Token:        ✅ Expires in 47min (weight 2)
canTrade:     ✅ canTrade=True (weight 3)
Balance:      ⚠️ Drawdown 47% (weight 1, score 0.6)
Latency:      ✅ P95=312ms (weight 1)

Trust Score = (3×1.0 + 2×1.0 + 3×1.0 + 1×0.6 + 1×1.0) / (3+2+3+1+1) × 100
            = 9.6 / 10.0 × 100
            = 96.0  → HEALTHY
```

### What happens when trust drops

1. Trust Score falls below 80 → **WARNING** alert
2. Trust Score falls below 50 → **CRITICAL** alert → stop trading
3. Trust Score recovers above 80 → **RECOVERY** notification

---

## Quick Start

```bash
git clone https://github.com/nessos666/topstepx-api-health-monitor.git
cd topstepx-api-health-monitor

pip install -r requirements.txt

cp .env.example .env
# Edit .env with your TopStepX credentials

python api_health_scanner.py
```

---

## Output

Results go to `/tmp/nq_api_health.json` (configurable):

```json
{
  "timestamp": "2026-03-21T10:30:00+00:00",
  "trust_score": 92.0,
  "status": "HEALTHY",
  "checks": [
    {"name": "reachability", "passed": true, "score": 1.0, "detail": "HTTP 200 in 245ms"},
    {"name": "latency", "passed": true, "score": 1.0, "detail": "get_accounts: p95=312ms"},
    {"name": "data_freshness", "passed": true, "score": 1.0, "detail": "Last bar 42s ago"},
    {"name": "contract", "passed": true, "score": 1.0, "detail": "CON.F.US.MNQ.M26 valid"},
    {"name": "token", "passed": true, "score": 1.0, "detail": "Expires in 47min"},
    {"name": "can_trade", "passed": true, "score": 1.0, "detail": "canTrade=True"},
    {"name": "bar_quality", "passed": true, "score": 1.0, "detail": "All bars valid"},
    {"name": "balance", "passed": true, "score": 0.8, "detail": "Drawdown 12.4%"},
    {"name": "loop_continuity", "passed": true, "score": 1.0, "detail": "No gaps detected"}
  ],
  "alerts": []
}
```

**Plain JSON.** Read it with anything — n8n, cron, Grafana, Prometheus, your own scripts.

---

## Alerting

The scanner does **one thing well**: check and report. It writes JSON to disk. You plug in the alerting.

### Option 1: n8n webhook (recommended)

```
[n8n Webhook Node] ← reads JSON ← [Scanner writes /tmp/nq_api_health.json]
         ↓
[IF trust_score < 80] → [Telegram Bot] → "⚠️ API DEGRADED: Score 73"
[IF trust_score < 50] → [Telegram Bot] → "🚨 CRITICAL: Score 42 — STOP TRADING"
[IF trust_score > 80 after alert] → [Telegram Bot] → "✅ RECOVERED: Score 91"
```

### Option 2: Plain cron

```bash
*/5 * * * * python /path/to/api_health_scanner.py && \
  python -c "
import json
d = json.load(open('/tmp/nq_api_health.json'))
if d['trust_score'] < 50:
    print('CRITICAL - stopping trades')
    # your stop-trading logic here
  "
```

### Option 3: Grafana

```ini
# datasource: JSON API
# dashboard: gauge showing trust_score
# alert: when value < 80 for 2 consecutive checks
```

---

## Latency Trending

Every run appends to `logs/latency_history.csv`:

```csv
timestamp,endpoint,p50,p95,min,max,avg,count
2026-03-21T10:30:00,get_accounts,280,312,180,450,290,5
2026-03-21T10:35:00,get_accounts,290,350,200,480,310,5
```

Track API performance degradation over days and weeks. Catch problems before they hit your trades.

---

## Running 24/7 (systemd)

```ini
# ~/.config/systemd/user/nq-apihealth.service
[Unit]
Description=TopStepX API Health Monitor

[Service]
ExecStart=/path/to/venv/bin/python /path/to/api_health_scanner.py
Restart=always
RestartSec=10
EnvironmentFile=/path/to/.env

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now nq-apihealth
```

---

## Configuration

All via environment variables or `.env`:

| Variable | Default | Description |
|----------|---------|-------------|
| `PROJECTX_USERNAME` | *required* | Your TopStepX username |
| `PROJECTX_API_KEY` | *required* | Your TopStepX API key |
| `PROJECTX_ACCOUNT_ID` | *required* | Account ID to monitor |
| `PROJECTX_CONTRACT_ID` | `CON.F.US.MNQ.M26` | Futures contract |
| `HEALTH_OUTPUT_FILE` | `/tmp/nq_api_health.json` | Where to write results |
| `HEALTH_CHECK_INTERVAL` | `300` | Seconds between checks |
| `STARTING_BALANCE` | `50000` | For drawdown calculation |

---

## Market Hours

Automatically handles NQ futures schedule:
- **Trading**: Sunday 18:00 – Friday 17:00 ET
- **Daily pause**: 17:00 – 18:00 ET
- Data-dependent checks (freshness, bar quality) auto-pass when market is closed — no false alerts on weekends.

---

## Contract Rollover Detection

Futures contracts expire quarterly. Miss the rollover and your algo trades a dead contract.

Check #4 knows the schedule:
- **H** (March) → **M** (June) → **U** (September) → **Z** (December)
- Rollover = 2nd Friday of the expiry month
- Alerts you before it happens, not after.

---

## Included: TopStepX API Client

`topstep_api.py` is a standalone, production-grade API client:

- JWT auth with automatic renewal
- Market orders with bracket (SL + TP)
- Per-endpoint latency tracking (P50, P95)
- Retry with exponential backoff
- Contract rollover validation
- Simulation mode (no real orders without `PROJECTX_LIVE_TRADING=1`)

Use it as a library in your own projects.

---

## Project Structure

```
├── api_health_scanner.py     # Main scanner — 9 checks, trust score, JSON output
├── topstep_api.py            # Production-grade TopStepX API client
├── requirements.txt
├── .env.example              # Environment variables template
└── README.md
```

---

## Testing

```bash
# Syntax check
python3 -m py_compile api_health_scanner.py
python3 -m py_compile topstep_api.py
```

---

## Related

Part of the trading infrastructure ecosystem:

- [rithmic-api-health-monitor](https://github.com/nessos666/rithmic-api-health-monitor) — Same approach for Rithmic API (file-based, no own connection)
- [api-health-trust-system](https://github.com/nessos666/api-health-trust-system) — Generic version of the Trust Score system for any REST API
- [tv-watch-agent](https://github.com/nessos666/tv-watch-agent) — 24/7 TradingView chart surveillance via CDP

---

## Philosophy

This tool exists because **trading infrastructure should be open**. The big firms have monitoring dashboards. Retail algo traders deserve the same.

If this saves you from one bad trade caused by a silent API failure, it paid for itself.

---

## Authors

- **[nessos666](https://github.com/nessos666)** – Creator, algo trader
- **Claude Bobby 1** – AI co-developer

Built as a human-AI team. Fair credit where it's due.

## License

MIT — Use it, modify it, share it. No strings attached.

<p align="center">
  <small>Built by an algo trader who got burned by silent API failures.<br>
  <strong>github.com/nessos666</strong></small>
</p>
