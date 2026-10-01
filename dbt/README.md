# weather_analytics (dbt project)

Transforms the raw Open-Meteo weather data loaded by the Airflow ETL DAG into daily weather metrics for Portland and Austin.

## Data flow

~~~
DEMO_DB.RAW.WEATHER_TEMPERATURE    (loaded by Airflow)
  -> ANALYTICS.STG_WEATHER         (view: cleaned and renamed columns)
  -> ANALYTICS.WEATHER_METRICS     (table: calculated metrics)
  -> ANALYTICS.WEATHER_SNAPSHOT    (snapshot: change history)
~~~

## Models

| Model | Type | Description |
|---|---|---|
| stg_weather | view | Renames columns, adds daily average temperature |
| weather_metrics | table | One row per city per day with the metrics below |

## Metrics

| Column | Meaning |
|---|---|
| temp_7d_moving_avg | Average temperature over the current day and previous 6 days |
| temp_anomaly | Daily average temperature minus the city's average over the loaded period |
| rainfall_7d_rolling_mm | Total precipitation over the current day and previous 6 days |
| dry_spell_length | Consecutive dry days (precipitation below 1 mm) up to the current day |


### How dry spell length is calculated

Done in three steps in `weather_metrics.sql`, per city, ordered by date:

1. **Flag dry days:** `is_dry_day` is 1 when precipitation is below 1 mm, otherwise 0.
2. **Group the days:** a running count of rainy days (`wet_day_count`) goes up by 1 on each rainy day. Each rainy day starts a new group, and the dry days after it share that group number.
3. **Count within each group:** a running sum of `is_dry_day` inside each group gives 1, 2, 3... for consecutive dry days. Rainy days are set to 0.

Example:

| Day | Rain (mm) | is_dry_day | wet_day_count | dry_spell_length |
|---|---|---|---|---|
| 1 | 0.0 | 1 | 0 | 1 |
| 2 | 0.0 | 1 | 0 | 2 |
| 3 | 2.4 | 0 | 1 | 0 |
| 4 | 0.0 | 1 | 1 | 1 |
| 5 | 0.0 | 1 | 1 | 2 |


## Tests

- not_null on key columns of both models
- assert_unique_city_date: one row per city per date
- assert_rainfall_not_negative: precipitation is never negative
- assert_temp_max_above_min: daily max is never below daily min

## Snapshot

weather_snapshot tracks changes to temperature, precipitation and weather code per city and date using the check strategy.

## How to run

Requires a dbt profile named weather_analytics in ~/.dbt/profiles.yml pointing to Snowflake.

~~~
dbt run
dbt test
dbt snapshot
~~~
