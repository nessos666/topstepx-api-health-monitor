# TopStepX API Health Monitor

Real-time health monitoring for the TopStepX / ProjectX futures trading API. Runs 9 automated checks every 5 minutes and outputs a **Trust Score (0-100)**.

Built for algo traders who need to know if their API connection is healthy before placing orders.

## The 9 Checks

| # | Check | What it does |
|---|-------|-------------|
| 1 | **Reachability** | Is the API responding? (HTTP 200) |
| 2 | **Latency** | Per-endpoint P95 latency < 2000ms? |
| 3 | **Data Freshness** | Last bar < 5 minutes old? |
| 4 | **Contract** | Is the futures contract still valid? (auto-rollover detection) |
| 5 | **Token** | Is the JWT token still valid? (auto-renewal) |
| 6 | **canTrade** | Is the account allowed to trade? |
| 7 | **Bar Quality** | No NaN values, volume > 0, OHLC plausible? |
| 8 | **Balance** | Drawdown < 80%? |
| 9 | **Loop Continuity** | Are your live scanners running without gaps? |

## Trust Score

- **80-100**: HEALTHY - All systems go
- **50-79**: DEGRADED - Some checks failing, investigate
- **0-49**: CRITICAL - Do not trade, fix issues first

## Quick Start

```bash
# Clone
git clone https://github.com/nessos666/topstepx-api-health-monitor.git
cd topstepx-api-health-monitor

# Setup
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your TopStepX credentials

# Run
python api_health_scanner.py
```

## Output

The scanner writes results to `/tmp/nq_api_health.json` (configurable):

```json
{
  "timestamp": "2026-03-21T10:30:00+00:00",
  "trust_score": 92.0,
  "status": "HEALTHY",
  "checks": [
    {"name": "reachability", "passed": true, "score": 1.0, "detail": "HTTP 200 in 245ms"},
    {"name": "latency", "passed": true, "score": 1.0, "detail": "get_accounts: p95=312ms"},
    ...
  ],
  "alerts": [],
  "latency": {
    "get_accounts": {"p50": 280, "p95": 312, "min": 180, "max": 450, "avg": 290, "count": 5}
  }
}
```

## Alerting

The scanner itself does **not** send alerts. It writes JSON that external tools can read:

- **n8n**: HTTP Request node reads the JSON, sends Telegram/email on low trust
- **systemd timer**: Runs a script that checks trust_score
- **cron**: `*/5 * * * * python check_trust.py`

This keeps the scanner simple and the alerting flexible.

## Latency Trending

Every check appends to `logs/latency_history.csv`:

```csv
timestamp,endpoint,p50,p95,min,max,avg,count
2026-03-21T10:30:00,get_accounts,280,312,180,450,290,5
2026-03-21T10:30:00,get_bars,450,620,380,750,490,5
```

Use this for long-term API performance analysis.

## Configuration

All settings via environment variables or `.env` file:

| Variable | Default | Description |
|----------|---------|-------------|
| `PROJECTX_USERNAME` | required | TopStepX username |
| `PROJECTX_API_KEY` | required | TopStepX API key |
| `PROJECTX_ACCOUNT_ID` | required | Account ID to monitor |
| `PROJECTX_CONTRACT_ID` | `CON.F.US.MNQ.M26` | Contract to check |
| `PROJECTX_LIVE_TRADING` | `0` | Not needed for monitoring |
| `HEALTH_OUTPUT_FILE` | `/tmp/nq_api_health.json` | Output path |
| `HEALTH_CHECK_INTERVAL` | `300` | Check interval in seconds |
| `STARTING_BALANCE` | `50000` | For drawdown calculation |

## Market Hours

The scanner automatically detects NQ futures market hours:
- **Open**: Sunday 18:00 ET - Friday 17:00 ET
- **Daily pause**: 17:00 - 18:00 ET
- Checks that depend on live data (freshness, bar quality) auto-pass when market is closed

## Contract Rollover

Automatic detection of quarterly contract rollover:
- H (March) → M (June) → U (September) → Z (December)
- Rollover happens on the 2nd Friday of the quarterly month
- Check #4 will alert if your configured contract is wrong

## Running as a Service (systemd)

```ini
# /etc/systemd/user/nq-apihealth.service
[Unit]
Description=TopStepX API Health Monitor

[Service]
ExecStart=/path/to/venv/bin/python /path/to/api_health_scanner.py
Restart=always
RestartSec=10
Environment=PROJECTX_USERNAME=your_user
Environment=PROJECTX_API_KEY=your_key
Environment=PROJECTX_ACCOUNT_ID=12345678

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now nq-apihealth
```

## Included: TopStepX API Module

`topstep_api.py` is a standalone API client for TopStepX with:

- JWT authentication with auto-renewal
- Market orders with bracket (SL + TP)
- Position management
- Per-endpoint latency tracking (P50, P95)
- Retry with exponential backoff
- Contract rollover validation
- Simulation mode (no real orders without `PROJECTX_LIVE_TRADING=1`)

## License

MIT
