# Weather Prediction Analytics

This project collects daily weather data for Portland and Austin using the Open-Meteo API, loads it into Snowflake through Airflow, transforms it with dbt, and visualizes the results in Preset.

## Architecture

Open-Meteo API → Airflow ETL → Snowflake RAW → dbt transformations → Preset dashboard

## Cities

- Portland, Oregon: 45.5234, -122.6762
- Austin, Texas: 30.2672, -97.7431

The Airflow pipeline retrieves the previous 60 days of weather data for both cities.

## Technologies

- Docker
- Apache Airflow
- PostgreSQL
- Snowflake
- dbt
- Preset
- Open-Meteo API

## Snowflake Raw Table

`DEMO_DB.RAW.WEATHER_TEMPERATURE`

Columns:

- `CITY`
- `LATITUDE`
- `LONGITUDE`
- `DATE`
- `TEMP_MAX`
- `TEMP_MIN`
- `PRECIPITATION`
- `WEATHER_CODE`

## dbt Output Objects

- `DEMO_DB.ANALYTICS.STG_WEATHER`
- `DEMO_DB.ANALYTICS.WEATHER_METRICS`
- `DEMO_DB.ANALYTICS.WEATHER_SNAPSHOT`

The analytics model calculates:

- Seven-day moving average temperature
- Temperature anomaly
- Seven-day rolling rainfall
- Dry spell length

## Airflow DAGs

### WeatherTemperatureETL

This DAG:

1. Extracts weather data from Open-Meteo
2. Transforms the API response into rows
3. Loads the data into Snowflake

The load uses `BEGIN`, `DELETE`, `INSERT`, and `COMMIT`. If an error occurs, the transaction is rolled back.

### WeatherAnalyticsDBT

This DAG runs after the ETL DAG and executes:

1. `dbt run`
2. `dbt test`
3. `dbt snapshot`

The workflow is:

`WeatherTemperatureETL → dbt run → dbt test → dbt snapshot`

## Airflow Configuration

The project uses:

- Airflow connection: `snowflake_conn`
- Airflow Variable: `weather_cities`

The dbt project is mounted inside the Airflow container at:

`/opt/airflow/dbt`

The Airflow Web UI is available at:

`http://localhost:8081`

## Data Quality

The dbt project includes tests for:

- Non-negative rainfall
- Maximum temperature above minimum temperature
- Unique city and date combinations
- Required fields not being null

The dbt snapshot tracks historical changes to weather records.

## Dashboard

Built in Preset (hosted Apache Superset) on `DEMO_DB.ANALYTICS.WEATHER_METRICS`, the table created by dbt.

**Charts**
- Daily Temperature vs 7-Day Moving Average
- Daily Temperature Anomaly by City
- Daily and 7-Day Rolling Rainfall by City
- Dry Spell Length by City

**Filters:** City and Date range. Metrics are precomputed in dbt, so filters change what is shown, not how the metrics are calculated.

![Full dashboard](screenshots/01_dashboard_full.png)

![Dashboard filtered to Portland, September](screenshots/06_dashboard_filtered_portland_sept.png)

Screenshots of each chart are in the [screenshots](screenshots/) folder. The dashboard is in a private Preset workspace, so screenshots are included instead of a link.

## Security

Private keys, passwords, passphrases, and local environment files are excluded from the repository. Snowflake access is configured through Airflow connections and dbt environment variables.
