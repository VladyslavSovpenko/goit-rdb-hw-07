# goit-rdb-hw-07 — часові ряди та напівструктуровані дані: NYC Taxi, JSONB, GIN

## Джерело даних

**NYC TLC Trip Record Data**, файл `yellow_tripdata_2024-03.parquet` (березень 2024, 3.58 млн поїздок, 60 МБ):
https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-03.parquet (офіційна сторінка: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page).

Файл завантажується кодом у ноутбуці (`urlretrieve`) і **не додається до репозиторію**. Stub-schema fallback не використано — реальні дані завантажились.

**Вибірка:** кожен 7-й рядок файлу (`iloc[::7]`, ≈ 512 тис. поїздок) замість `head(500_000)`, щоб покрити всі 31 день місяця (перші 500 тис. рядків — лише перші ~4 дні, і 30-денні лаги були б порожніми). Очищення за умовою: `0 < fare_amount < 500`, `0 < trip_distance < 100`, `PULocationID` не NULL, лише березень 2024. Після очищення — 491 308 поїздок → `nyc_taxi_raw` → daily-агрегація `t7_hw_taxi_daily` (5 415 рядків: 248 зон × до 31 дня, PK `(pickup_zone_id, ts_day)`).

## Time-series features (усі вікна з `PARTITION BY pickup_zone_id`)

| Ознака | Вікно |
|---|---|
| `trips_lag_1d / 7d / 30d` | `LAG(trip_count, k) OVER (PARTITION BY pickup_zone_id ORDER BY ts_day)` |
| `rolling_avg_fare_7d` | `RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW` (7 календарних днів включно з поточним) |
| `rolling_std_fare_30d` | `STDDEV_SAMP` у `RANGE BETWEEN INTERVAL '29 days' PRECEDING` |
| `cumulative_trips`, `expanding_avg_trips` | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

Об'єднано у view `v_t7_hw_taxi_features` (фільтр `trips_lag_30d IS NOT NULL`).

## JSONB-частина

`t7_hw_events` — 50 000 синтетичних подій 5 типів (`page_view`, `click`, `add_to_cart`, `checkout`, `recommendation_shown`) з різними payload. Запити: containment `@>`, key existence `?` / `?|`, JSONPath (`jsonb_path_query`, `jsonb_path_exists`). GIN `jsonb_path_ops` та expression B-tree `((payload ->> 'device'))` з `EXPLAIN (ANALYZE, BUFFERS)` до/після; нормалізована `t7_hw_checkouts_norm` і порівняння одного запиту у двох версіях; MongoDB-equivalent як pseudocode у markdown.

## Як запустити

**Google Colab:** відкрити `notebooks/hw7_taxi_timeseries_jsonb.ipynb`, *Runtime → Restart and run all*. PostgreSQL 16 стартує через `pgserver` (`/tmp/hw7_pg`), Parquet завантажується у `/content/`.

**Локально з PostgreSQL 16** (наприклад, Docker `postgres:16`):

```bash
export HW7_PG_URI='postgresql+psycopg2://user:password@localhost:5432/dbname'
jupyter notebook notebooks/hw7_taxi_timeseries_jsonb.ipynb
```
