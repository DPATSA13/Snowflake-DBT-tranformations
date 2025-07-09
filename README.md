# Snowflake + dbt: MovieLens Analytics Pipeline (End-to-End)

This document provides a **complete, structured, and unskipped end-to-end flow** for building a **Snowflake + dbt pipeline** to transform MovieLens CSV data from **raw S3 ingestion to final fact tables**, aligned with Medallion architecture for your **Data Architect learning and GitHub portfolio**.

---

## 1️⃣ Environment Setup in Snowflake

### Create Warehouse, Database, Schema

```sql
USE ROLE ACCOUNTADMIN;
CREATE ROLE IF NOT EXISTS TRANSFORM;
GRANT ROLE TRANSFORM TO ROLE ACCOUNTADMIN;

CREATE WAREHOUSE IF NOT EXISTS COMPUTE_WH;
GRANT OPERATE ON WAREHOUSE COMPUTE_WH TO ROLE TRANSFORM;

CREATE DATABASE IF NOT EXISTS MOVIELENS;
CREATE SCHEMA IF NOT EXISTS MOVIELENS.RAW;

GRANT ALL ON WAREHOUSE COMPUTE_WH TO ROLE TRANSFORM;
GRANT ALL ON DATABASE MOVIELENS TO ROLE TRANSFORM;
GRANT ALL ON ALL SCHEMAS IN DATABASE MOVIELENS TO ROLE TRANSFORM;
```

### Create User for dbt

```sql
CREATE USER IF NOT EXISTS dbt PASSWORD = '****'
DEFAULT_WAREHOUSE = 'COMPUTE_WH'
DEFAULT_ROLE = TRANSFORM
DEFAULT_NAMESPACE = 'MOVIELENS.RAW';

GRANT ROLE TRANSFORM TO USER dbt;
```

### Create External Stage for S3 Ingestion

```sql
CREATE STAGE netflixstage
URL = 's3://netflixpatsa'
CREDENTIALS = (AWS_KEY_ID='***' AWS_SECRET_KEY='***');
```

---

## 2️⃣ Ingesting CSV Data from S3 into Snowflake Staging Tables

### Create staging tables and load data:

```sql
CREATE OR REPLACE TABLE raw_movies (
    movieId INTEGER,
    title STRING,
    genres STRING
);
COPY INTO raw_movies FROM '@netflixstage/movies.csv' FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

CREATE OR REPLACE TABLE raw_ratings (
    userId INTEGER,
    movieId INTEGER,
    rating FLOAT,
    timestamp BIGINT
);
COPY INTO raw_ratings FROM '@netflixstage/ratings.csv' FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

CREATE OR REPLACE TABLE raw_tags (
    userId INTEGER,
    movieId INTEGER,
    tag STRING,
    timestamp BIGINT
);
COPY INTO raw_tags FROM '@netflixstage/tags.csv' FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"') ON_ERROR = 'CONTINUE';

CREATE OR REPLACE TABLE raw_genome_tags (
    tagId INTEGER,
    tag STRING
);
COPY INTO raw_genome_tags FROM '@netflixstage/genome-tags.csv' FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

CREATE OR REPLACE TABLE raw_genome_scores (
    movieId INTEGER,
    tagId INTEGER,
    relevance FLOAT
);
COPY INTO raw_genome_scores FROM '@netflixstage/genome-scores.csv' FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');

CREATE OR REPLACE TABLE raw_links (
    movieId INTEGER,
    imdbId INTEGER,
    tmdbId INTEGER
);
COPY INTO raw_links FROM '@netflixstage/links.csv' FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"');
```

✅ Validate row counts for each table to confirm ingestion.

---

## 3️⃣ Setting Up dbt Locally

### Install dbt using `pipx`:

```bash
pipx install dbt-snowflake
pipx ensurepath
```

Add `~/.local/bin` to your PATH in `~/.zshrc`:

```bash
export PATH="$PATH:/Users/your_username/.local/bin"
source ~/.zshrc
```

### Initialize the dbt Project

```bash
dbt init my_dbt_project
```

### Configure `~/.dbt/profiles.yml`:

```yaml
my_dbt_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      account: your_account
      user: dbt
      password: your_password
      role: TRANSFORM
      database: MOVIELENS
      warehouse: COMPUTE_WH
      schema: RAW
      threads: 1
```

✅ Validate with:

```bash
dbt debug
```

---

## 4️⃣ Create `dbt_project.yml`

```yaml
name: 'my_dbt_project'
version: '1.0.0'
profile: 'my_dbt_project'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

clean-targets:
  - "target"
  - "dbt_packages"

models:
  my_dbt_project:
    +materialized: view

    staging:
      +materialized: view

    dim:
      +materialized: table

    fct:
      +materialized: table
```

---

## 5️⃣ Building the dbt Pipeline

### Create and populate `models/staging/` with `src_` models (views):

* `src_movies.sql`, `src_ratings.sql`, `src_tags.sql`, `src_genome_tags.sql`, `src_genome_scores.sql`, `src_links.sql` referencing `MOVIELENS.RAW` tables.

✅ Run:

```bash
dbt run
```

✅ Validate creation of `SRC_*` views.

---

### Create and populate `models/dim/` with `dim_` models (tables):

**Example `dim_movies.sql`:**

```sql
{{ config(materialized='table') }}

WITH src_movies AS (
    SELECT * FROM {{ ref('src_movies') }}
)
SELECT
    movie_id,
    INITCAP(TRIM(title)) AS movie_title,
    SPLIT(genres, '|') AS genre_array,
    genres
FROM src_movies
```

✅ Repeat similarly for `dim_ratings.sql`, `dim_tags.sql`, `dim_links.sql`, `dim_genome_tags.sql`, `dim_genome_scores.sql` using `{{ ref('src_*') }}`.

✅ Run:

```bash
dbt run
```

✅ Validate creation of `DIM_*` tables.

---

### Create and populate `models/fct/` with `fct_` models (fact tables):

**Example `fct_user_activity.sql`:**

```sql
{{ config(materialized='table') }}

WITH ratings AS (
    SELECT DISTINCT user_id FROM {{ ref('dim_ratings') }}
),
tags AS (
    SELECT DISTINCT user_id FROM {{ ref('dim_tags') }}
)
SELECT DISTINCT user_id
FROM (
    SELECT * FROM ratings
    UNION
    SELECT * FROM tags
)
```

✅ Run:

```bash
dbt run
```

✅ Validate creation of `FCT_*` fact tables.

---

## 6️⃣ Testing and Documentation

✅ Create tests in `tests/` for `not_null` and `unique` constraints on primary keys.
✅ Run:

```bash
dbt test
```

✅ Generate documentation and explore lineage:

```bash
dbt docs generate
dbt docs serve
```

---

## ✅ Final Verification Checklist:

✅ Snowflake configured with raw data ingested from S3.
✅ dbt environment installed, debugged, and connected.
✅ `staging` models (`SRC_*`) created as views.
✅ `dim` models (`DIM_*`) created as tables.
✅ `fct` models (`FCT_*`) created as tables.
✅ Tests passing.
✅ Lineage documented and visualized.
✅ Ready for ML and dashboard integrations.

---

## 🚀 Next Steps

* Create Power BI / Streamlit dashboards using `FCT_*` tables.
* Integrate GitHub Actions for CI/CD on your dbt project.
* Extend with incremental models for large-scale data pipelines.

---

This structured end-to-end pipeline prepares you to **showcase your Snowflake + dbt skills confidently for your Data Architect or Senior Data Engineer roles** while reinforcing systematic architecture practices for your real-world learning.
# Snowflake-DBT-tranformations
