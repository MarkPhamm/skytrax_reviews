# Skytrax Global Airlines Analytics Project

<img width="2000" height="1333" alt="image" src="https://github.com/user-attachments/assets/95ba2599-7690-4a35-981e-99dc9704ee40" />

---

## Repositories

| Repository | Owner | Purpose |
| --- | --- | --- |
| **[skytrax\_extract\_load](https://github.com/MarkPhamm/skytrax_reviews_extract_load)** | MarkPhamm | EL pipeline: scrapes 160K+ airline reviews, stages to S3, loads into Snowflake via Airflow |
| **[skytrax\_transformation](https://github.com/MarkPhamm/skytrax_reviews_transformation)** | MarkPhamm | dbt transformation into star schema with slim CI/CD, IaC, and hosted dbt docs |
| **[skytrax\_data\_cleaning](https://github.com/DucLe-2005/british_airways_data_cleaning)** | DucLe-2005 | Cleans raw scraped data and standardizes formats using modular Python functions |
| **[skytrax\_dashboard\_website](https://github.com/nguyentienTCU/Skytrax_Reviews_Dashboard)** | nguyentienTCU | Dashboard website for visualising insights from processed airline reviews |
| **[spirit\_airlines\_dashboard](https://github.com/MiaTran1112/spirit_airlines_dashboard)** | MiaTran1112 | Mode dashboard analyzing Spirit Airlines customer satisfaction |

---

## Team

### Leadership

* **Mentor / Stakeholder:** [Nhan Tran](https://www.linkedin.com/in/panicpotatoe/)
* **Team Lead / Analytics Engineer:** [Mark Pham](https://www.linkedin.com/in/minhbphamm/)

### Members

* **Data Analysts:** [Trang Dam](https://www.linkedin.com/in/thuytrangdam/), [Gwennie Nguyen](https://www.linkedin.com/in/gwennienguyen/), [Jenny Tran](https://www.linkedin.com/in/jennytranuyen/), [Mia Tran](https://www.linkedin.com/in/miatran1207/), [Alyssa Le](https://www.linkedin.com/in/alyssaqle/)
* **Data Engineers:** [Leonard Dau](https://www.linkedin.com/in/leonard-dau-722399238/), [Thieu Nguyen](https://www.linkedin.com/in/thieunguyen1402/), [Viet Lam Nguyen](https://www.linkedin.com/in/lam-nguyen-viet-051a57305)
* **Software Engineers:** [Tien Nguyen](https://www.linkedin.com/in/tien-nguyen-598758329), [Anh Duc Le](https://www.linkedin.com/in/duc-le-517420205/)
* **Data Scientists:** [Robin Tran](https://www.linkedin.com/in/robin-tran/), [Trung Dam](https://www.linkedin.com/in/trung-dam-86962a235/)
* **Scrum Master:** [Hien Dinh](https://www.linkedin.com/in/hiendinhq)

---

## Project Overview

An end-to-end analytics pipeline that ingests, transforms, and visualises **customer review data for every airline on Skytrax** (AirlineQuality.com). The project scrapes 160,000+ reviews, loads them into Snowflake, transforms them into a Kimball star schema with dbt, and serves insights through an interactive dashboard.

> **Self-selection bias:** Skytrax reviews are self-reported. Passengers with extreme experiences are more likely to post, so KPIs skew away from the broader flying population. The goal is *directional insight*, not population-level generalisation.

---

## Architecture

```text
airlinequality.com
        | scrape (26 A-Z parallel tasks)
        v
S3: raw/YYYY/MM/raw_data_YYYYMMDD.csv
        | clean + transform
        v
S3: processed/YYYY/MM/clean_data_YYYYMMDD.csv
        | COPY INTO
        v
Snowflake: SKYTRAX_REVIEWS_DB.RAW.AIRLINE_REVIEWS
        | dbt source
        v
staging -> intermediate -> marts (star schema)
        |
        v
Dashboard / BI Tools / RAG Chatbot
```

## Stack

| Layer | Technology |
| --- | --- |
| Scraping | Python 3.12, BeautifulSoup, pandas |
| Orchestration | Apache Airflow (Astronomer Runtime, Docker) |
| Storage | AWS S3 (date-partitioned landing zone) |
| Data Warehouse | Snowflake |
| Transformation | dbt (dbt-snowflake), SQLFluff |
| IaC | Terraform (AWS S3/IAM/CloudFront/OIDC + Snowflake RBAC/schemas/warehouses) |
| CI/CD | GitHub Actions (slim CI with merge-base, defer/favor-state CD) |
| Auth | AWS IAM OIDC (keyless GitHub Actions) |
| Docs | dbt docs hosted on CloudFront + S3 |
| Dashboard | Next.js, TailwindCSS, Chart.js, LangChain, ChromaDB |

---

## Part 1: Extract & Load

**Repo:** [skytrax\_reviews\_extract\_load](https://github.com/MarkPhamm/skytrax_reviews_extract_load)

### Pipeline

Three Airflow DAGs chained via Airflow Datasets (no cron, no polling):

| DAG | Trigger | What it does |
| --- | --- | --- |
| `skytrax_crawl` | Daily 02:00 UTC | Scrapes reviews with 26 parallel A-Z tasks, uploads raw CSVs to S3 |
| `skytrax_process` | Dataset (raw) | Downloads raw CSVs, cleans/transforms, uploads processed CSVs |
| `skytrax_snowflake` | Dataset (processed) | Runs `COPY INTO` Snowflake for each review date |

### Loading Strategy

* **Incremental (daily):** scrapes only yesterday's reviews. Each review date maps to exactly one CSV, so re-runs are idempotent.
* **Bulk backfill:** trigger with `full_scrape=True` to scrape all historical reviews (back to 2010). Snowflake's `COPY INTO` tracks loaded files — no duplicates.

### S3 Layout

```text
s3://skytrax-reviews-landing-<account-id>/
  raw/YYYY/MM/raw_data_YYYYMMDD.csv
  processed/YYYY/MM/clean_data_YYYYMMDD.csv
```

* Versioning enabled, AES256 encryption, lifecycle rules (Standard-IA after 30d, expire old versions after 90d), all public access blocked.

### Data Cleaning

Column standardisation (snake_case), ISO 8601 dates, text cleaning, route parsing (origin/destination/connections), aircraft name normalisation, and numeric rating conversion.

---

## Part 2: Transformation & CI/CD

**Repo:** [skytrax\_reviews\_transformation](https://github.com/MarkPhamm/skytrax_reviews_transformation)

### Data Model (Star Schema)

Follows Kimball methodology with deterministic surrogate keys. **Grain:** one row per customer review per flight.

| Model | Type | Description |
| --- | --- | --- |
| `fct_review` | Fact | Review metrics, ratings, calculated averages, rating bands |
| `dim_customer` | Dimension | Reviewer identity and flight count |
| `dim_airline` | Dimension | Airline name |
| `dim_aircraft` | Dimension | Aircraft model, manufacturer, seat capacity |
| `dim_location` | Dimension | City + airport (role-playing: origin, destination, transit) |
| `dim_date` | Dimension | Calendar + fiscal dates (role-playing: submitted, flown) |

**[Live dbt Docs](https://d38l3fc9bckvbz.cloudfront.net)** — auto-generated and hosted on CloudFront, updated on every deploy.

### Snowflake Infrastructure

All managed by Terraform — users, roles, grants, schemas, warehouses. No manual setup.

| Schema | Purpose |
| --- | --- |
| `RAW` | Raw source data from EL pipeline |
| `SOURCE` | Staging views — 1:1 source mirrors |
| `INTERMEDIATE` | Cleaned/normalized business logic |
| `MARTS` | Star schema dims + facts |
| `STAGING` | CI scratch space |
| `DEV_*` | Per-user dev schemas for local development |

### CI/CD

* **Continuous Integration (PRs):** merge-base state comparison — only changed models are linted (SQLFluff), compiled, run, and tested.
* **Continuous Deployment (merge to `main`):** `dbt build --select state:modified+ --defer --favor-state` rebuilds only modified models + downstream. Uploads manifest, run_results, and dbt docs to S3. Invalidates CloudFront cache.
* **Keyless auth:** GitHub Actions authenticates to AWS via OIDC — no static credentials.

---

## Part 3: Visualisation

**Repo:** [skytrax\_dashboard\_website](https://github.com/nguyentienTCU/Skytrax_Reviews_Dashboard) | **[Live Dashboard](https://british-airways-dashboard-website.vercel.app/)**

* **Interactive KPI Cards:** overall satisfaction, NPS-like scores, category averages
* **Multi-Dimensional Filters:** airline, aircraft, route, cabin class, traveller type
* **Data Explorer:** drag-and-drop or SQL-like querying for power users
* **RAG Chatbot:** natural-language Q&A across the full corpus of reviews

---

## Key Business Insights

### Economy-Class Trends

![Economy Insights](https://github.com/MarkPhamm/British-Airway/assets/99457952/665ff202-218a-4862-a130-98ce4c8584b9)

* Ground-staff service and boarding efficiency dominate complaints across airlines
* Major hubs (LHR, CDG, JFK) see the highest negative volume — tied to long queues and staff shortages
* 92% of low-rating Economy reviews cite *at-airport* factors rather than in-flight experience

![Economy Recommendations](https://github.com/MarkPhamm/British-Airway/assets/99457952/fad27d46-f9c1-4187-94af-02da65d3f10b)

**Recommendations:** boost ground-handling staffing during peak waves, deploy self-service kiosks and real-time queue monitoring.

### Premium-Cabin Expectations

* Business & First passengers focus on seat comfort, bedding quality, and connectivity speed
* Consistency gaps between aircraft sub-fleets drive dissatisfaction
* Food quality is the second-largest driver of sub-4-star ratings

**Recommendations:** accelerate fleet-wide seat upgrades, introduce chef-curated rotating menus, guarantee minimum bandwidth per passenger.

---

## Next Steps

1. **Expand Data Sources** — integrate on-time performance and DOT complaint data
2. **Real-Time Ingestion** — CDC-style pipelines to surface insights within hours of review publication
3. **Predictive Modelling** — sentiment + operational variables to forecast NPS by airline and route
4. **Monetisation** — benchmarking dashboards for airlines and airports via subscription

---

*© 2025 Skytrax Global Airlines Analytics Project*
