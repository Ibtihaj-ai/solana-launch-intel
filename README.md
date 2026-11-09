# Solana Fresh Launch Intel — MVP (Railway)

Read-only data collector + small dashboard. **No trading, no wallets, no keys.**

What runs (one service + one Postgres):

- **PumpPortal** (free stream, one shared WebSocket): every new Pump.fun token + migration event → saved to `raw_events` and `tokens`. Auto-reconnect with backoff, duplicates ignored, bounded queue with backpressure (nothing dropped).
- **DexScreener** snapshots (batch of 30 mints per request, ~200 req/min max): price, liquidity, market cap, FDV, volume, txns → `market_snapshots`. PumpSwap/other pools discovered here go to `pools` (pool age kept separate from token age).
- **Collect first, filter later**: nothing is dropped for low liquidity / market cap. Filters (1,000 liquidity, 800k–1B mcap, 20 min age) are only dashboard views.
- Dashboard: All / Newest / Candidates / Rejected (with exact reason) / Migrated, token detail with charts, provider health.

## Deploy on Railway

1. Push this folder to a GitHub repo → Railway → **New Project → Deploy from GitHub repo**.
2. In the same project: **+ New → Database → PostgreSQL**.
3. Open the app service → **Variables** → add:
   - `DATABASE_URL` = `${{Postgres.DATABASE_URL}}`
   - `DASHBOARD_PASSWORD` = a long random password (dashboard asks for it; username can be anything)
   - `APP_TIMEZONE` = `Asia/Karachi` (optional, default already)
4. **Settings → Networking → Generate Domain**. Open it, log in. Tables are created automatically on first start.

Healthcheck is `/healthz`. Data collection starts as soon as the service is up.

## Storage warning (important)

Pump.fun launches tens of thousands of tokens per day. Default schedule (20s → 60s → 3m → 10m → 30m over 24h) writes roughly 90 snapshot rows per token that has a DexScreener pair. Watch **DB size** on the dashboard. To grow slower, set `SNAPSHOT_SCALE=2` (or 3) — all intervals are multiplied. `MONITOR_ALL_TOKENS_HOURS` sets how long each token is tracked (default 24). `MONITOR_ALL_TOKENS_EXTENDED_DAYS=7` adds hourly snapshots up to 7 days (much more data).

## Limits / not in this MVP

- Discovery is Pump.fun + migrations only. Raydium-native pools are discovered only if DexScreener returns them for a tracked token. Dedicated Raydium decoding comes later.
- Token creation time = the moment we receive the event (live stream), not a chain timestamp. Tokens first seen at migration are marked `stub` (age unknown, not used as age).
- No trade stream (that is metered). Volume/txns are DexScreener-reported values, stored as reported.
- If the service is down, events during that time are missed (PumpPortal is live-only, no history).
- In-memory queue: events already queued but not yet written can be lost on a hard crash (graceful stop drains it).
- Health counters reset on restart; data does not.
- No wallet/holder/risk analysis, outcomes, backtests or filter builder yet.

## Local run

```bash
pip install -r requirements.txt
export DATABASE_URL=postgresql://user:pass@localhost:5432/intel
uvicorn app.main:app --reload
python -m pytest tests
```

Set `RUN_COLLECTOR=false` to run the dashboard only.
