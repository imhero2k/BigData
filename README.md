# FIT5202 – Data Processing for Big Data (2025 S2)

**Assignments Portfolio: A1 · A2A · A2B**
Monash University — Sai Kandala Sesha Maruthi Sathya (authcate: `xxxx0000`)

> This repository contains my end‑to‑end work for FIT5202 across all three assignments:
>
> 1. **A1**: Historical analysis of the Australian property market (RDDs, DataFrames, Spark SQL).
> 2. **A2A**: Feature engineering + ML pipelines for building energy forecasting.
> 3. **A2B**: Real‑time streaming + ML predictions + visualisation (Kafka + Spark Structured Streaming).

---

## 🔖 Badges & Tech

* Python **3.11**
* PySpark **3.5.x**
* Kafka **3.x** (KRaft or ZK)
* JupyterLab / Notebook
* macOS (Apple Silicon) / Linux
* Timezone: **Australia/Melbourne** (Spark: `spark.sql.session.timeZone = Australia/Melbourne`)

---

## 📁 Repository Structure

```
.
├── A1_property/
│   ├── A1_<authcate>.ipynb
│   └── data/                 # nsw_property_price.csv, council.json, zoning.json, property_purpose.json
│
├── A2A_energy_model/
│   ├── A2A_<authcate>.ipynb
│   ├── models/               # saved Spark ML pipelines (best model, tuned model)
│   └── data/                 # buildings.csv, meters.csv, weather.csv
│
├── A2B_streaming/
│   ├── Assignment-2B-Task1_producer_<authcate>.ipynb
│   ├── Assignment-2B-Task2_spark_streaming_<authcate>.ipynb
│   ├── Assignment-2B-Task3_consumer_<authcate>.ipynb
│   ├── config.yaml           # central config (Kafka brokers, topics, paths, TZ)
│   ├── checkpoints/          # Spark checkpoints (streaming)
│   ├── out_parquet/          # parquet sinks (6h building agg; daily site agg)
│   └── kafka/                # helper scripts (start, create-topics)
│
├── requirements.txt          # minimal Python deps to run notebooks
├── Makefile                  # convenience commands
└── README.md
```

> **Note:** Replace `<authcate>` with your Monash authcate ID in filenames before submission.

---

## 🚀 Quickstart

### 1) System Prereqs

* Python 3.11 (`brew install python@3.11` on macOS).
* Java 8/11 (for Spark).
* Apache Spark 3.5.x (or use the course Docker image).
* Kafka 3.x (local or Docker).

### 2) Python Env

```bash
# From repo root
/opt/homebrew/bin/python3.11 -m venv .venv311
source .venv311/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

**`requirements.txt` (excerpt):**

```
pyspark==3.5.0
pyarrow
pandas
matplotlib
plotly
kafka-python   # or confluent-kafka
python-dotenv  # optional
```

### 3) Spark Timezone

I use Melbourne time everywhere:

```python
from pyspark.sql import SparkSession
spark = (SparkSession.builder
    .appName("FIT5202-Notebooks")
    .config("spark.sql.session.timeZone", "Australia/Melbourne")
    .getOrCreate())
```

---

## 🧩 Assignment 1 – Property Market (Historical Analysis)

**Due:** 23:55 Fri **05/Sep/2025** (Week 6) — **Weight 15%**

**Notebook:** `A1_property/A1_<authcate>.ipynb`

### Goals

* Practice Spark RDDs, DataFrames, Spark SQL on the NSW property dataset.
* Transformations, joins, partitioning strategies, and performance comparisons.

### Highlights

* **RDD**: load CSV/JSON, clean invalid records, custom partitioning, UDF to map ISO dates → `DD/Mon/YYYY`.
* **DF API**: unit normalisation (area → sqm), standardise purpose categories, multi‑sale value gains, seasonal price trends, your own exploration.
* **Comparison**: implement one complex query in two approaches (RDD vs DF or DF vs SQL), time them, and discuss.

### Submission

* `A1_<authcate>.ipynb`
* `A1_<authcate>.pdf` (export with outputs)

---

## 🤖 Assignment 2A – Energy Model (Feature + ML)

**Due:** 23:55 Fri **26/Sep/2025** (Week 9) — **Weight 15%**

**Notebook:** `A2A_energy_model/A2A_<authcate>.ipynb`

### Goals

* Engineer features and build **Spark ML** pipelines (Random Forest, GBT) to predict aggregated building energy consumption.

### Pipeline (at a glance)

1. **Aggregations (label):** per‑building energy every **6 hours** (0–5:59, 6–11:59, 12–17:59, 18–23:59).
2. **Imputation:** weather nulls → `Imputer(strategy="mean")`.
3. **Seasonality:** derive **peak/off‑peak** flag per site using the top‑3 hottest and coldest months.
4. **VectorAssembler/Indexers:** build features from buildings + weather.
5. **Models:** **RF** and **GBT** estimators.
6. **Metric:** implement custom **RMSLE** (Root Mean Squared Log Error).
7. **Tuning:** `ParamGridBuilder` + `CrossValidator` (or `TrainValidationSplit`).
8. **Persist best model:** `A2A_energy_model/models/best_pipeline/` (Spark `PipelineModel.write().overwrite().save(path)`).

### Submission

* `A2A_<authcate>.ipynb`
* `A2A_<authcate>.pdf` (with outputs)

---

## ⚡ Assignment 2B – Streaming + ML + Viz

**Due:** 23:55 Fri **24/Oct/2025** (Week 12) — **Weight 15%**
**Demo:** Week 14 (compulsory)

**Notebooks:**

* `A2B_streaming/Assignment-2B-Task1_producer_<authcate>.ipynb`
* `A2B_streaming/Assignment-2B-Task2_spark_streaming_<authcate>.ipynb`
* `A2B_streaming/Assignment-2B-Task3_consumer_<authcate>.ipynb`

### Architecture

```
[weather.csv] --(producer/5s)--> [Kafka: weather5s]
                                   |
                        [Spark Structured Streaming]
                          |  - watermark(weather_ts, 5s)
                          |  - feature transforms (reuse 2A)
                          |  - load 2A model -> predictions
                          +--> parquet streams: out_parquet/6h_by_building
                          +--> parquet streams: out_parquet/daily_by_site
                                   |
                            [streaming parquet readers]
                                   |
                         [Kafka: energy_6h_by_building]
                         [Kafka: energy_daily_by_site]
                                   |
                   [consumer/viz -> plots + shortfall/excess]
```

### Default Kafka Topics

* `weather5s` (producer → weather batches every 5 seconds, 5 days per tick, `weather_ts` added & spread across 5 seconds)
* `energy_6h_by_building` (Task 2 — 7s trigger; print 20 records)
* `energy_daily_by_site` (Task 2 — 14s trigger)

### Running (local Kafka example)

```bash
# 1) Start Kafka (choose one)
#    a) Docker compose (recommended) — see A2B_streaming/kafka/README.md
#    b) Native Kafka: start-kafka.sh (ZK or KRaft)

# 2) Create topics
kafka-topics --create --topic weather5s --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092
kafka-topics --create --topic energy_6h_by_building --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092
kafka-topics --create --topic energy_daily_by_site --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092

# 3) Open notebooks in order
jupyter lab
#   - Task1 producer: streams weather5s (hourly -> 5 days per batch, add weather_ts)
#   - Task2 spark_streaming: reads Kafka, transforms, loads A2A model, writes parquet + Kafka
#   - Task3 consumer: reads 6b/6c topics + meters; draws plots + shortfall/excess
```

### Spark Streaming Notes (Task 2)

* **Session**: 4 cores, app name set, timezone **Australia/Melbourne**, **checkpoint** configured.
* **Schema**: keep `weather_ts` as **Int**; parse others from String.
* **Watermark**: discard records **>5s late**.
* **Triggers**:

  * **7s** → aggregate 6‑hour energy per building; display **20** rows.
  * **14s** → aggregate **daily** energy per site.
* **Sinks**: write both aggregates to **Parquet** (streaming mode, append) and also to **Kafka**.

### Visualisation (Task 3)

* Load **new_meters.csv** (metered ground truth).
* Plot two diagrams for **6b** (building/6h) and **6c** (site/daily).
* Plot **shortfall/excess** per site/day: *(predicted total − metered total)* — can be ±.

---

## ⚙️ Config (`A2B_streaming/config.yaml`)

```yaml
kafka:
  bootstrap_servers: "localhost:9092"
  topics:
    weather_in: "weather5s"
    agg_6h_by_building: "energy_6h_by_building"
    agg_daily_by_site: "energy_daily_by_site"

spark:
  app_name: "FIT5202-A2B-Streaming"
  timezone: "Australia/Melbourne"
  checkpoint_dir: "A2B_streaming/checkpoints"
  out_parquet_dir: "A2B_streaming/out_parquet"

model:
  path: "A2A_energy_model/models/best_pipeline"

paths:
  weather_csv: "A2A_energy_model/data/weather.csv"
  buildings_csv: "A2A_energy_model/data/buildings.csv"
  new_buildings_csv: "A2B_streaming/data/new_building_information.csv"
  new_meters_csv: "A2B_streaming/data/new_meters.csv"
```

---

## 🧪 Reproducibility & Performance

* Random seeds fixed where applicable; keep versions pinned in `requirements.txt`.
* Use `%time`/`%%time` in notebooks to log stage runtimes (especially A1 Part 3 & A2A model training).
* For large datasets, prefer DF API over RDDs; persist intermediate frames judiciously.

---

## 🧭 Makefile (optional)

Common tasks (examples):

```makefile
venv:
	/opt/homebrew/bin/python3.11 -m venv .venv311
	. .venv311/bin/activate && pip install --upgrade pip -r requirements.txt

jlab:
	. .venv311/bin/activate && jupyter lab

kafka-create-topics:
	kafka-topics --create --topic weather5s --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092
	kafka-topics --create --topic energy_6h_by_building --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092
	kafka-topics --create --topic energy_daily_by_site --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092
```

---

## 🧾 Academic Integrity & AI Usage

* This repository reflects my own work.
* **Generative AI** was used **selectively** for brainstorming and refactoring; all prompts/attributions are listed in `A2A_energy_model/REFERENCES.md` and `A2B_streaming/REFERENCES.md` (per unit policy).
* Please see the **Monash academic integrity** policy and unit instructions. Do not share your private solutions.

---

## 🗓️ Due Dates (for reference)

* **A1**: 23:55 **Fri 05/Sep/2025** (Week 6)
* **A2A**: 23:55 **Fri 26/Sep/2025** (Week 9)
* **A2B**: 23:55 **Fri 24/Oct/2025** (Week 12) — **Demo in Week 14**

---

## 🧰 Demo Checklist (A2B)

* ✅ Kafka up, topics created; producer emits `weather5s` every 5s.
* ✅ Spark streaming running with checkpointing, prints 6b (7s) & 6c (14s) outputs.
* ✅ Parquet sinks updating in `out_parquet/*`.
* ✅ Consumer plots (6b, 6c) and shortfall/excess.
* ✅ Back‑up screenshots; have a brief 1‑2 min architecture walkthrough ready.

---

## 📄 License

MIT for **this repository’s code** only. Datasets are provided by the unit and **must not** be re‑distributed.

---

## 🙋 Contact

* Sai K S M Sathya — `xxxx0000`
* Email: [your.email@monash.edu](mailto:your.email@monash.edu)

> If you’re marking this repo: open each notebook top‑down; outputs rendered; environment notes at top of each notebook.
