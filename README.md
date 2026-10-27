# Egypt Air Quality Data Platform

An automated data pipeline that collects air quality and weather data for Egyptian cities, validates and cleans it, models it in a PostgreSQL star schema, and serves it through Grafana dashboards and Telegram alerts.

> **Status:** in development. See the [Roadmap](#roadmap).

## Problem

Air pollution in Cairo and the Nile Delta (including the autumn "black cloud" season) is a public health issue, but air quality data is scattered across APIs with different formats, units, and update rates. This project builds a reliable, repeatable pipeline that turns those sources into one trusted dataset and makes it easy to see which areas are worst, when, and why.

## Data Sources

| Source | Type | Used for |
|---|---|---|
| OpenAQ API | Station measurements | Ground-truth readings where stations exist |
| Open-Meteo Air Quality API | Modeled estimates | PM2.5, PM10, NO2, O3, SO2, CO for all selected cities |
| Open-Meteo Weather API | Modeled estimates | Wind, humidity, temperature as context |
| ESP32 + PM sensor (optional) | Streaming via MQTT | Extra local source for comparison |

**Important:** Open-Meteo air quality values are model-based estimates, not station measurements. Where OpenAQ stations are available, both are stored side by side so the difference can be analyzed. OpenAQ coverage in Egypt must be verified before relying on it.

## Architecture

![Architecture](docs/architecture.png)

```
APIs -> raw -> validate (+ rejected records) -> transform (AQI, weather join) -> star schema -> Grafana + Telegram alerts
```

Everything runs on one machine with Docker Compose: Airflow, PostgreSQL, Grafana, and (optionally) Mosquitto.

## Tech Stack

- **Orchestration:** Apache Airflow
- **Processing:** Python, Pandas, SQL
- **Storage:** PostgreSQL (schemas: `raw`, `staging`, `dw`, `ops`)
- **Visualization and alerts:** Grafana, Telegram bot
- **Infrastructure:** Docker Compose
- **Optional:** ESP32, MQTT (Mosquitto)

## Data Model

Star schema in the `dw` schema:

- `fact_air_quality_hourly`: city, hour, source, pollutant values, AQI, AQI category, wind speed, humidity, temperature
- `dim_location`: city, governorate, latitude, longitude
- `dim_source`: OpenAQ, Open-Meteo, sensor
- `dim_date`: date and hour attributes
- `dim_aqi_category`: AQI ranges and labels

Supporting tables in `ops`: `rejected_records`, `dq_results`, `pipeline_runs`.

## Pipeline (Airflow DAG)

`extract -> load_raw -> validate -> transform -> load_dw -> quality_check -> alert`

- Runs hourly, with retries and logging on every task
- A backfill DAG loads historical data for trend analysis

## Data Quality Rules

- Required fields not null (location, timestamp, pollutant values)
- Values within plausible ranges (no negative concentrations)
- No duplicate (location, timestamp, source) records
- Timestamps standardized to a single timezone
- Freshness check: alert if no new data arrives within the expected window
- Invalid records are stored in `ops.rejected_records` with the rejection reason, never silently dropped

## Project Structure

```
egypt-air-quality-pipeline/
├── dags/            # Airflow DAGs
├── src/             # extract, validate, transform modules
├── sql/             # schema, views, analysis queries
├── docs/            # architecture diagram, ERD, screenshots
├── docker-compose.yml
├── .env.example
└── README.md
```

## Getting Started

```bash
git clone https://github.com/<your-username>/egypt-air-quality-pipeline.git
cd egypt-air-quality-pipeline
cp .env.example .env      # fill in your own keys
docker compose up -d
```

Environment variables (`.env.example`):

```
OPENAQ_API_KEY=
POSTGRES_USER=
POSTGRES_PASSWORD=
POSTGRES_DB=
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
```

Never commit your real `.env` file.

## Analysis Examples

- Worst 5 cities by average PM2.5 this month
- Hour-of-day pollution pattern per city
- Relationship between wind speed and PM2.5
- Modeled vs measured differences where stations exist

## Roadmap

- [ ] Verify API coverage for selected cities
- [ ] Docker Compose environment
- [ ] Raw ingestion and historical backfill
- [ ] Validation and rejected-records table
- [ ] Transform, AQI calculation, star schema
- [ ] Airflow DAG end to end
- [ ] Grafana dashboard
- [ ] Telegram alerts
- [ ] Documentation, ERD, and demo screenshots
- [ ] Optional: ESP32 sensor via MQTT

## Limitations

- Open-Meteo pollutant data is modeled, not measured.
- Low-cost sensors need calibration against a reference source.
- Station coverage in Egypt may be limited.

## Author

Your name, Computer Science, Lotus University
