# Revenue per Customer (Square seller GPV per day)

**Best table to use:** `app_support.app_support.seller_daily_merchant`

## Grain

One row per `MERCHANT_TOKEN` × `REPORT_DATE`. Square-only (joins through `vdim_merchant` — does not include Cash, Afterpay, etc.).

## Columns to use for GPV

- **`GPV_GROSS_FXD_USD`** — daily GPV in fixed USD. Use this for cross-currency totals.
- `GPV_GROSS_BASE_UNIT` — daily GPV in the merchant's local currency.
- `ANNUALIZED_GPV` = `SUM(gpv_fxd_usd_last_91d) * 4`.

Currency conversion is already handled upstream in `app_bi.hexagon.vagg_daily_processing_summary`.

## Important caveats

1. **GPV lags 2 days.** Rows exist for today and yesterday, but `GPV_GROSS_FXD_USD` is 0 until the upstream `vagg_daily_processing_summary` backfills. Always filter `WHERE REPORT_DATE <= CURRENT_DATE - 2` for fully-populated GPV.
2. **Not every Square seller is in this table.** The underlying user-days seed filters to merchants with `case_tokens` — i.e. sellers who have ever opened a support case. If you need GPV for the full Square seller universe, go directly to `app_bi.hexagon.vagg_daily_processing_summary` instead.
3. Excludes `user_type = 'merchant'` rows (units, not parent merchants); unit-level GPV is rolled up to the merchant token.

## Lineage

```
seller_daily_merchant
 ← app_support.seller_daily_user                      (unit grain, same ETL)
     ← app_bi.hexagon.vagg_daily_processing_summary   ← TERMINAL: GPV source
     ← app_bi.hexagon.vagg_daily_revenue_summary      ← TERMINAL: AR source
     ← app_bi.hexagon.vdim_date / dim_report_period
 ← app_bi.hexagon.vdim_user, vdim_merchant
 ← app_support.intermediates.seller_business_identity
```

## Build / refresh

- **Not a dbt model.** Built by a legacy Snowflake ETL.
- SQL: [`old_square_etls/seller_daily/seller_daily.sql`](https://github.com/squareup/app-cash-cs/blob/main/old_square_etls/seller_daily/seller_daily.sql#L1117-L1474) in `squareup/app-cash-cs` (lines 1117–1474).
- Schedule: daily at 02:00 UTC (`cron: 0 2 * * *`), warehouse `ETL__LARGE`.
- Owner: AVINASHSHEKAR.
- Build pattern: rebuild last-week staging on Fridays, incremental staging for newer days, UNION ALL → swap rename to final table.
