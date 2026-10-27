# Undertow sample: crypto liquidations and liquidation maps

A free sample from [Undertow](https://liq.adalia.ee/?s=gh) by Adalia Holding OÜ:
BTC, ETH and SOL, 2026-10-14 to 2026-10-20 (UTC).

- **Liquidations per minute** from seven venues in one schema: Binance, Bybit, OKX
  (exchanges) and Hyperliquid, Jupiter, GMX, dYdX (on-chain perpetuals). Each row says
  whether the venue publishes every liquidation or only a sample.
- **Point-in-time liquidation maps**: the first map of every UTC hour, every price bin,
  with the sampled layer (the on-chain positions of the accounts we read) and the modeled
  layer (inferred from open interest) kept apart, and the model version on every row.

The week includes 2026-10-15, when $444M of BTC, ETH and SOL liquidations were recorded
on these venues, $105M of it in the minute starting 02:28 UTC.

## Download

The files are in the [release](../../releases/latest): one archive with everything, and
the two Parquet files on their own. Check them against `SHA256SUMS`.

| File | Rows | What it holds |
| --- | --- | --- |
| `liquidations_1m.parquet` / `.csv` | 14,595 | Per minute, venue, symbol, side: prints and USD, with coverage and price basis |
| `map_snapshots_hourly.parquet` / `.csv` | 88,322 | 504 hourly map snapshots (3 symbols, 168 hours), 172 to 176 bins each |
| `venue_coverage.csv` | 7 | Per venue for the week: coverage, price basis, prints, USD |
| `connection_events.csv` | 174 | Disconnects and reconnects of our Binance, Bybit and OKX collectors |
| `README.md` | | The full data dictionary |

## Load it

```python
import pandas as pd

liq = pd.read_parquet("liquidations_1m.parquet")
maps = pd.read_parquet("map_snapshots_hourly.parquet")
liq.groupby(["venue", "side"])["notional_usd"].sum()
```

```sql
-- DuckDB
SELECT minute_utc, venue, side, sum(notional_usd) AS usd
FROM 'liquidations_1m.parquet'
WHERE minute_utc >= '2026-10-15 02:20:00+00' AND minute_utc < '2026-10-15 02:30:00+00'
GROUP BY ALL ORDER BY minute_utc, usd DESC;
```

## What to know before using it

- **Coverage.** Binance publishes at most one liquidation per contract per second, so its
  totals are lower bounds. Our OKX and Hyperliquid collection is sampled too. Bybit,
  Jupiter, GMX and dYdX are full. `coverage` marks this on every row.
- **Sides.** `side` is the side of the liquidated position. In the hours when price fell
  by 0.5% or more, long liquidations were 91 to 100% of each venue's USD; in the hours
  when it rose by 0.5% or more, 0 to 7%.
- **A model change inside the sample.** Between the 13:00 and 14:00 UTC maps of
  2026-10-19, the weight of the sampled layer in the blend changed: from 13:27 UTC
  Hyperliquid's open interest is counted at one side. `exact_fraction` for BTC went from
  0.1317 to 0.0707 with open interest unchanged.
  The model version id is the same on both sides, so the change is marked only in the
  data dictionary.
- **Point in time.** Each map is stamped with `as_of` (the time it describes) and
  `fetched_at` (when it was written). Use `fetched_at` to avoid look-ahead.
- **The map is a model.** Its modeled layer is inferred, not observed. No accuracy claim
  is made: the history is there for you to test it.

## The rest of the data

- Live maps for twelve coins by API, paid per call in USDC over x402: 0.01 USDC per coin,
  0.08 USDC for all twelve. No account needed. See [liq.adalia.ee](https://liq.adalia.ee/?s=gh).
- The full liquidation records, the map history and Hyperliquid positions with their
  liquidation prices: on request, with an evaluation sample, while their API endpoints
  are built.
- Contact: [@adaliaundertow](https://x.com/adaliaundertow) on X, or the contact details
  on [liq.adalia.ee](https://liq.adalia.ee/?s=gh).

## License

Free for evaluation and research; not to be redistributed as a dataset. See
[LICENSE](LICENSE). Not investment advice.
