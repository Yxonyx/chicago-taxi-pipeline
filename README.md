# Chicago Taxi + Weather ETL (AWS Lambda + S3)

End‑to‑end, serverless data pipeline that ingests Chicago taxi trip data and same‑day hourly weather, stages raw JSON in S3, transforms to analytics‑ready CSVs, and maintains small “master” lookup tables (company and payment type).

> **Repository**: https://github.com/Yxonyx/chicago-taxi-pipeline  
> **Main Notebook**: `Lamdba_full.ipynb` (Transform + Load)  
> **Cloud**: AWS Lambda, Amazon S3, CloudWatch Logs

---

## 1) Architecture Overview

```
Chicago Open Data (Socrata)        Open‑Meteo ERA5 API
            │                                  │
            └─────────────── Extract Lambda ───┘      (runs daily or on demand)
                                 │
                                 ▼
                    s3://cubixchicagodata/rawdata/to_processed/
                       taxi_data/taxi_raw_YYYY-MM-DD.json
                       weather_data/weather_raw_YYYY-MM-DD.json
                                 │  (S3 Put event)
                                 ▼
                         Transform + Load Lambda
                                 │
                                 ├─ Clean + normalize taxi
                                 │    • drop heavy geoms, unify column names
                                 │    • create `datetime_for_weather` (hour-rounded)
                                 │    • grow master tables (company, payment_type)
                                 │    • attach master IDs back to facts
                                 │
                                 ├─ Flatten weather hourly arrays
                                 │
                                 └─ Write outputs + move raw
                                      s3://.../transformed_data/taxi_trips/taxi_YYYY-MM-DD.csv
                                      s3://.../transformed_data/weather/weather_YYYY-MM-DD.csv
                                      s3://.../transformed_data/payment_type/payment_type_master.csv
                                      s3://.../transformed_data/company/company_master.csv

                                      move raw JSON to:
                                      s3://.../rawdata/processede/{taxi_data|weather_data}/...
```

**Why this setup?**  
- Clear “inbox” (`rawdata/to_processed`) → transformation → “archive” (`rawdata/processede`) keeps reprocessing + recursion risks low.  
- Small master tables keep company/payment strings normalized with stable IDs.  
- Weather and taxi timestamps are aligned hourly to allow simple joins downstream.

---

## 2) S3 Layout (Bucket: `cubixchicagodata`)

**Incoming (Extract Lambda writes here):**
```
rawdata/to_processed/taxi_data/     taxi_raw_YYYY-MM-DD.json
rawdata/to_processed/weather_data/  weather_raw_YYYY-MM-DD.json
```

**Processed (Transform Lambda moves raw JSON here):**
```
rawdata/processede/taxi_data/       taxi_raw_YYYY-MM-DD.json
rawdata/processede/weather_data/    weather_raw_YYYY-MM-DD.json
```

**Transformed CSV outputs:**
```
transformed_data/taxi_trips/        taxi_YYYY-MM-DD.csv
transformed_data/weather/           weather_YYYY-MM-DD.csv
transformed_data/payment_type/      payment_type_master.csv
transformed_data/company/           company_master.csv
transformed_data/master_table_previous_version/
                                    payment_type_master_previous_version.csv
                                    company_master_previous_version.csv
```

> Note: the folder name uses `processede` by design to match the existing structure in this project.

---

## 3) Lambdas

### 3.1 Extract Lambda (Taxi + Weather → S3)

- **Taxi API**: `https://data.cityofchicago.org/resource/ajtu-isnz.json` (Socrata)  
  SOQL example (single day window, UTC):  
  ```sql
  select *
  where trip_start_timestamp between 'YYYY-MM-DDT00:00:00' and 'YYYY-MM-DDT23:59:59'
  limit 30000
  ```
  Optional `X-App-Token` header from env: `CHICAGO_KEY` or `CHICAGO_API_TOKEN`.

- **Weather API**: Open‑Meteo ERA5 (hourly)  
  Parameters include the day, Chicago lat/lon (41.85, -87.65) and hourly variables:
  `temperature_2m, wind_speed_10m, rain, precipitation`.

- **Target day**: `UTC now - 2 months`, same calendar day (to mirror the instructor’s setup).

- **Output**:
  - `rawdata/to_processed/taxi_data/taxi_raw_YYYY-MM-DD.json`
  - `rawdata/to_processed/weather_data/weather_raw_YYYY-MM-DD.json`

- **Logging**: prints counts like `TAXI page 1: +13929` and confirms uploads.

### 3.2 Transform + Load Lambda (S3 → CSV + Masters)

- **Trigger**: S3 `ObjectCreated:*` events on **one** of these filter strategies:
  - **Recommended**: *Single rule* with prefix `rawdata/to_processed/` and suffix `.json`  
    (avoids “overlapping suffix/prefix” conflicts), **or**
  - Two separate rules, one per prefix (`taxi_data/` and `weather_data/`) with `.json` suffix, ensuring they **do not** overlap for the same Lambda.

- **Recursion guard**: On S3 events, the handler checks that keys start with `rawdata/to_processed/`. If not, it exits early to prevent loops. This is critical because the Lambda itself writes to `transformed_data/` and moves files to `rawdata/processede/`.

- **What it does**:
  1. **Taxi**:  
     - Drops heavy, unused columns  
       (`pickup/dropoff_census_tract`, `pickup/dropoff_centroid_location`)  
     - Removes fully empty rows  
     - Renames community areas to `_id` suffix if needed  
     - Adds `datetime_for_weather = floor(trip_start_timestamp, 1h)`  
     - **Master tables**: reads current `payment_type_master.csv` and `company_master.csv`, appends any new values with incremental IDs, and writes back (archiving the previous version).  
     - Merges `payment_type_id` and `company_id` back into the taxi DataFrame.  
     - Writes `transformed_data/taxi_trips/taxi_YYYY-MM-DD.csv`  
     - Moves the raw JSON to `rawdata/processede/taxi_data/…`
  2. **Weather**:  
     - Flattens hourly arrays to a table: `datetime, temperature, wind_speed, rain, precipitation`  
     - Writes `transformed_data/weather/weather_YYYY-MM-DD.csv`  
     - Moves the raw JSON to `rawdata/processede/weather_data/…`

- **Edge case**: Some taxi days return `{"error": ..., "message": ...}` via Socrata. The Lambda treats these as placeholders:
  - **Still moves** the raw JSON to clear the inbox.
  - Writes an empty CSV only if you decide to keep a complete date footprint; otherwise skip writing transformed for that day.

- **CloudWatch logs** include helpful lines like:
  - `TAXI columns: [...]`
  - `OK -> transformed_data/taxi_trips/taxi_2025-07-05.csv`
  - `MOVED RAW -> rawdata/processede/taxi_data/taxi_raw_2025-07-05.json`
  - `payment_type_master updated.` / `company_master updated.`

---

## 4) Important Functions (Transform Lambda)

- `read_json_from_s3(bucket, key) -> Any`  
  Helper that replaces repetitive 3‑line patterns:
  ```python
  response = s3.get_object(Bucket=bucket, Key=key)
  content = response["Body"]
  data = json.loads(content.read())
  ```
  Use:
  ```python
  taxi_json    = read_json_from_s3(BUCKET, "rawdata/to_processed/taxi_data/taxi_raw_YYYY-MM-DD.json")
  weather_json = read_json_from_s3(BUCKET, "rawdata/to_processed/weather_data/weather_raw_YYYY-MM-DD.json")
  ```

- `taxi_trips_transformations(df: pd.DataFrame) -> pd.DataFrame`  
  Cleans and normalizes taxi rows; adds `datetime_for_weather`.

- `update_master(source_df, master_df, id_col, value_col) -> pd.DataFrame`  
  Adds unseen values to small lookup tables with incremental IDs.

- `attach_master_ids(taxi_df, payment_master, company_master) -> pd.DataFrame`  
  Left‑join IDs back into the taxi facts.

- `transform_weather_data(weather_json: dict) -> pd.DataFrame`  
  Flattens Open‑Meteo hourly measurements.

- `move_raw_and_write_transformed(...) -> Optional[str]`  
  Writes the CSV with a date suffix, then copies + deletes the raw JSON into `rawdata/processede/...`.

- Safety helpers:
  - `looks_like_error_payload(payload, df)` to detect `{"error","message"}` placeholders.
  - Recursion guard for S3 triggered events (ignore anything not under `rawdata/to_processed/`).

---

## 5) IAM Permissions (minimal)

Attach an execution role to both Lambdas with at least:
- `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` on `cubixchicagodata` and the listed prefixes.
- CloudWatch Logs: `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`.

The Extract Lambda also needs outbound internet access (default for Lambda in a public subnet or without VPC).

---

## 6) Environment Variables

**Extract Lambda**
```
BUCKET_NAME=cubixchicagodata
CHICAGO_KEY=<optional app token>       # or CHICAGO_API_TOKEN
```

**Transform Lambda**
*(currently the bucket name is hardcoded; you can move it to an env var for flexibility)*
```
BUCKET_NAME=cubixchicagodata   # if you refactor the code to read from env
```

---

## 7) Deployment & Triggers

1. **Create / reuse bucket** `cubixchicagodata` with the folder layout above.
2. **Create the Extract Lambda** and deploy the code. Test once — it will write today‑minus‑2‑months JSON files to the “to_processed” inbox.
3. **Create the Transform + Load Lambda** and set **one** S3 trigger:
   - **Option A (simple):**  
     - Event: `ObjectCreated:*`  
     - Prefix: `rawdata/to_processed/`  
     - Suffix: `.json`
   - **Option B (split):**  
     - Rule 1 → Prefix: `rawdata/to_processed/taxi_data/`, Suffix: `.json`  
     - Rule 2 → Prefix: `rawdata/to_processed/weather_data/`, Suffix: `.json`  
     Ensure you don’t create **overlapping** filters for the **same** Lambda and event type, otherwise S3 shows:  
     *“Configuration is ambiguously defined. Cannot have overlapping suffixes in two rules if the prefixes are overlapping for the same event type.”*

4. **Watch logs** in CloudWatch for both Lambdas.

---

## 8) Running Manually (Test Events)

- **Extract**: run a test with any dummy JSON; it computes the target day internally. It logs the target day and count of taxi rows/hours and writes the two raw JSON files.
- **Transform**: run a test with an empty `{}` event to process whatever is in `rawdata/to_processed/`. (With live S3 triggers, the Lambda is invoked automatically on each new JSON.)

---

## 9) Data Model (Key Columns)

### Taxi Facts (selected)
- `trip_id`, `taxi_id`
- `trip_start_timestamp`, `trip_end_timestamp`
- `trip_seconds`, `trip_miles`
- `pickup_community_area_id`, `dropoff_community_area_id`
- `fare`, `tips`, `tolls`, `extras`, `trip_total`
- `payment_type`, `company`
- `pickup_centroid_latitude`, `pickup_centroid_longitude`
- `dropoff_centroid_latitude`, `dropoff_centroid_longitude`
- **Derived:** `datetime_for_weather` (hour‑rounded start time)
- **Attached:** `payment_type_id`, `company_id` (from masters)

### Weather (hourly)
- `datetime`
- `temperature`
- `wind_speed`
- `rain`
- `precipitation`

### Masters
- `transformed_data/payment_type/payment_type_master.csv`:  
  `payment_type_id`, `payment_type`
- `transformed_data/company/company_master.csv`:  
  `company_id`, `company`

---

## 10) Operations, Cost & Safety

- **Recursion protection**: Transform ignores any event whose key doesn’t start with `rawdata/to_processed/`. This avoids Lambda → S3 → Lambda loops when the function writes output or moves files.
- **S3 event overlap**: prefer a single trigger, or ensure multiple triggers do not overlap on prefix/suffix for the **same** Lambda.
- **API edge days**: if Socrata returns `{"error","message"}`, the pipeline still **moves** the raw JSON so the inbox stays clean.
- **Pagination** (Extract): if a day exceeds `limit`, you can extend the Extract to page through more results.
- **Monitoring**: check CloudWatch logs; consider alarms on error rate or timeouts. You can also monitor the `RecursiveInvocationsDropped` metric (Lambda) if you ever change event triggers and want to ensure loops are not occurring.

---

## 11) Local Development

This repo keeps the full Transform + Load Lambda code in a Jupyter notebook: `Lamdba_full.ipynb`. For quick experiments locally you’ll need:
```
python 3.11+
pip install boto3 pandas
AWS credentials configured (to access the bucket)
```
> In a plain notebook environment without AWS creds, `boto3` calls will fail — deploy and test in AWS for end‑to‑end behavior.

---

## 12) Troubleshooting

- **“Repository not found” when pushing**: verify the remote URL and that your GitHub username/repo are correct.  
- **S3 trigger error**: *“Configuration is ambiguously defined … overlapping suffixes …”* → use a single trigger (prefix `rawdata/to_processed/`, suffix `.json`) or remove the overlapping one.  
- **Empty taxi CSV**: likely an API edge day or the raw JSON contained only `{error, message}`. Check the raw JSON in `rawdata/processede/taxi_data/` and CloudWatch logs.  
- **Recursive loop warning from AWS**: ensure the Transform Lambda is **only** triggered by the inbox prefixes and not by the transformed or processed prefixes.

---

## 13) Acknowledgments

- City of Chicago Open Data, Taxi Trips dataset (via Socrata)
- Open‑Meteo ERA5 hourly weather API
- AWS Lambda / Amazon S3 / CloudWatch

---

## 14) License

This project is provided for educational purposes. See the repository for license details (or add one, e.g., MIT).

