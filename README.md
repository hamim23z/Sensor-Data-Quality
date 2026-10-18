# Sensor Data Quality Framework

*A declarative framework that scores sensor telemetry for the failures that quietly corrupt fleet analytics, and publishes a daily quality scorecard per system.*

## Overview

Industrial fleets (solar, wind, any equipment reporting telemetry) generate sensor data that goes bad in predictable ways: a stuck sensor reports the same value for hours, a unit gets misconverted, a feed drops out for six minutes, a spike shows up from a bad read. Most analysis pipelines trust the data as-is, which means a single stuck sensor can quietly skew a fleet-wide average. This project is a check engine that catches those problems before they reach analysis: it profiles every sensor stream, runs a catalog of declarative checks, and rolls the results up into a daily per-system quality score.

**Status:** In development, built around a simulated fleet with injected, known faults, then validated against one real public dataset.

## What it checks

| Check | Detects |
| --- | --- |
| Flatline | A stuck sensor or frozen feed (value repeats while the system is running) |
| Spike | Single-sample outliers via rolling median/MAD |
| Range | Physically impossible values |
| Unit mismatch | A value distribution that jumps by a known conversion factor (e.g. °C read as °F) |
| Gap | Missing samples relative to expected cadence |
| Duplicate / out-of-order timestamps | Ingest bugs |
| Staleness | A feed that has stopped updating |
| Cross-signal consistency | Sensors that disagree with each other (e.g. power ≈ voltage × current) |
| Schema drift | A renamed, retyped, or missing column |

Checks are configured per sensor class in YAML, not code, so adding a new sensor type is a config change:

```yaml
tag_class: stack_temperature
unit: degC
cadence_seconds: 60
checks:
  - flatline: {window_samples: 30, min_std: 0.01, only_when: running}
  - range: {min: 20, max: 900}
  - spike: {method: rolling_mad, window_samples: 15, threshold: 6}
  - gap: {max_gap_multiple: 3}
```

## Tech stack

Python 3.12 · DuckDB · Polars/pandas · Pydantic · dbt-duckdb · Dagster · pytest + Hypothesis · Streamlit (scorecard UI) · Docker

## Data

- **`fleet-sim`** — a simulator (built for this project) that generates realistic telemetry for ~20 systems and deliberately injects faults with known ground truth, so detection can be measured, not assumed.
- **NREL PVDAQ** — one real, public solar dataset, used to confirm the checks hold up against genuinely messy real-world data, not just the simulator's own faults.

All data is synthetic or public. No employer, production, or proprietary fleet data is used anywhere in this project.

## Getting started

```bash
git clone https://github.com/<your-username>/fleet-data-quality.git
cd fleet-data-quality
docker compose up
```

This starts the simulator, runs the check pipeline via Dagster, and serves the scorecard app locally.

Run the checks against your own data instead:

```bash
python -m fleetdq.profile --input path/to/data.parquet --config configs/my_sensors.yaml
```

## Evaluation

Because `fleet-sim` records exactly which faults it injected and when, detection quality is measurable: the project reports precision and recall per fault type, rather than just asserting the checks "work."

## Project structure

```
fleet-data-quality/
  fleet_sim/        # synthetic fleet + fault injector
  checks/           # check engine (flatline, spike, range, ...)
  dbt/               # models turning flags into daily scores
  dagster_defs/      # orchestration
  app/                # scorecard UI
  configs/            # per-sensor-class YAML
  tests/
```
