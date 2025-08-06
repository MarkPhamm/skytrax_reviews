# Skytrax Global Airlines Analytics Project
<img width="2000" height="1333" alt="image" src="https://github.com/user-attachments/assets/95ba2599-7690-4a35-981e-99dc9704ee40" />


Access our Dashboard: [Global Airlines Dashboard](https://global-airlines-dashboard.vercel.app/)

---

## Repositories

| Repository                                                                                               | Owner         | Purpose                                                                                                                                |
| -------------------------------------------------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **[skytrax\_data\_cleaning](https://github.com/DucLe-2005/all_airlines_data_cleaning)**            | DucLe‑2005    | Cleans raw scraped data and standardizes formats using modular Python functions.                                                       |
| **[skytrax\_extract\_load](https://github.com/vietlam2002/all_airlines_extract_load)**             | vietlam2002   | Scrapes customer reviews for *all* airlines on Skytrax, stages them in S3, then loads to Snowflake via Airflow‑compatible ETL scripts. |
| **[skytrax\_transformation](https://github.com/MarkPhamm/all_airlines_transformation)**            | MarkPhamm     | Handles dbt‑based data transformation on Snowflake with CI/CD workflows via GitHub Actions.                                            |
| **[skytrax\_dashboard\_website](https://github.com/nguyentienTCU/all_airlines_dashboard_website)** | nguyentienTCU | A dashboard website for visualising insights from processed airline reviews.                                                           |
| **[skytrax\_analysis](https://github.com/trungdam512/all_airlines_analysis)**                      | trungdam512   | Focuses on EDA, ML models, and sentiment analysis across carriers.                                                                     |

---

## Team Structure

### Project Leadership

* **Mentor – Stakeholder:** [Nhan Tran](https://www.linkedin.com/in/panicpotatoe/)
* **Analytics Engineer – Team Lead:** [Mark Pham](https://www.linkedin.com/in/minhbphamm/)

### Engineering Teams

* **Data Engineering:** [Leonard Dau](https://www.linkedin.com/in/leonard-dau-722399238/), [Thieu Nguyen](https://www.linkedin.com/in/thieunguyen1402/), [Viet Lam Nguyen](https://www.linkedin.com/in/lam-nguyen-viet-051a57305)
* **Software Engineering:** [Tien Nguyen](https://www.linkedin.com/in/tien-nguyen-598758329), [Anh Duc Le](https://www.linkedin.com/in/duc-le-517420205/)
* **Data Science:** [Robin Tran](https://www.linkedin.com/in/robin-tran/), [Trung Dam](https://www.linkedin.com/in/trung-dam-86962a235/)
* **Scrum Master:** [Hien Dinh](https://www.linkedin.com/in/hiendinhq)

---

## 1. Project Overview

This end‑to‑end analytics initiative ingests, processes, and visualises **customer‑review data for *every* airline covered by Skytrax** (AirlineQuality.com). The architecture leverages industry‑standard tooling and cloud services to provide a robust, scalable foundation for airline‑wide sentiment, operational, and competitive analysis.

**Self‑selection bias:** Reviews on Skytrax are self‑reported. Passengers with extreme experiences (positive *or* negative) are more likely to post, so KPIs derived from this data skew away from the broader flying population. Our goal is therefore *directional insight*, not population‑level generalisation.

---

## 2. Architecture Overview

![BritishAirways](https://github.com/user-attachments/assets/2a9d45e6-be1b-4582-a9a0-3b7fb7536d9f)

### 2.1 Extraction Layer

#### Overview

The extraction layer gathers review data for *all* airlines from Skytrax, stores it in S3 and prepares it for downstream processing.

* **Repository:** [all\_airlines\_extract\_load](https://github.com/vietlam2002/all_airlines_extract_load)

#### 2.1.1 Technology Stack

* Python 3.12 with Pandas
* Apache Airflow
* AWS S3
* Docker
* Snowflake

#### 2.1.2 Data Source

Skytrax review pages, e.g.
`https://www.airlinequality.com/airline-reviews/{airline‑slug}/`

Captured fields include: star ratings, review text, flight details, passenger metadata, and category scores.

#### 2.1.3 Extraction Process

```python
# From main_dag.py – Extract task definition
scrape_skytrax_data = BashOperator(
    task_id="scrape_skytrax_data",
    bash_command="chmod -R 777 /opt/***/data && python /opt/airflow/tasks/scraper_extract/scraper.py"
)
```

Steps

1. Iterate through the Skytrax airline index.
2. Request paginated review HTML for each carrier.
3. Parse and normalise each review record.
4. Persist results to `raw_data.csv`.

#### 2.1.4 Data Cleaning & Initial Transformation

```python
clean_data = BashOperator(
    task_id="clean_data",
    bash_command="python /opt/airflow/tasks/transform/transform.py"
)
```

Cleaning tasks standardise date formats, handle nulls, and enforce data‑type consistency before staging to S3.

#### 2.1.5 AWS S3 Integration

```python
upload_cleaned_data_to_s3 = BashOperator(
    task_id="upload_cleaned_data_to_s3",
    bash_command="chmod -R 777 /opt/airflow/data && python /opt/airflow/tasks/upload_to_s3.py"
)
```

* Secure IAM roles
* Server‑side encryption
* Versioning enabled

#### 2.1.6 Workflow Orchestration

```python
with DAG(
    dag_id="skytrax_pipeline",
    schedule_interval="@daily",
    default_args=default_args,
    start_date=start_date,
    catchup=True,
    max_active_runs=1,
):
    scrape_skytrax_data >> note >> clean_data >> note_clean_data >> upload_cleaned_data_to_s3
```

#### 2.1.7 Snowflake Integration

```python
snowflake_copy_operator = BashOperator(
    task_id="snowflake_copy_from_s3",
    bash_command="pip install snowflake-connector-python python-dotenv && python /opt/airflow/tasks/snowflake_load.py"
)
```

---

### 2.2 Data Cleaning Layer

* **Repository:** [all\_airlines\_data\_cleaning](https://github.com/DucLe-2005/all_airlines_data_cleaning)
* **Stack:** Python 3.12.5, Pandas, NumPy, Matplotlib, Seaborn

Key steps mirror the British Airways version but operate across carriers:

1. **Column Standardisation** – snake\_case, special‑character cleanup.
2. **Date Formatting** – ISO 8601 for both submission and flight dates.
3. **Text Cleaning** – verification flag extraction; nationality normalisation.
4. **Route Parsing** – origin, destination, and connections.
5. **Aircraft Standardisation** – unified Airbus/Boeing nomenclature.
6. **Rating Conversion** – numeric Int64 fields for uniform analysis.

Outputs feed directly to Snowflake for transformation.

---

### 2.3 Transformation Layer

* **Repository:** [all\_airlines\_transformation](https://github.com/MarkPhamm/all_airlines_transformation)
* **Stack:** dbt (Core), Snowflake, Airflow (Astronomer), GitHub Actions

#### 2.3.1 Data Model

A star schema identical in design to the airline‑specific version:

| Table             | Purpose                                                 |
| ----------------- | ------------------------------------------------------- |
| **fct\_review**   | One row per review per flight with quantitative metrics |
| **dim\_customer** | Passenger information                                   |
| **dim\_aircraft** | Aircraft attributes                                     |
| **dim\_location** | Airport / city keys for origin, destination, transit    |
| **dim\_date**     | Calendar table for submission & flight dates            |

Incremental dbt jobs maintain freshness while minimising warehouse spend.

#### 2.3.2 Data Quality Framework

* Schema & relationship tests
* Custom business‑logic assertions (e.g. rating within 0–10)
* Freshness & completeness checks

CI/CD triggers on code pushes, PRs, weekly schedules, and manual invocations.

---

### 2.4 Visualisation Layer

* **Repository:** [all\_airlines\_dashboard\_website](https://github.com/nguyentienTCU/all_airlines_dashboard_website)
* **Live Site:** [Global Airlines Analytics Dashboard](https://global-airlines-dashboard.vercel.app/)
* **Stack:** Next.js, TailwindCSS, Chart.js, LangChain, ChromaDB

#### 2.4.1 Dashboard Highlights

* **Interactive KPI Cards:** Overall satisfaction, NPS‑like scores, category averages.
* **Multi‑Dimensional Filters:** Airline, aircraft, route, cabin class, traveller type.
* **Data Explorer:** Drag‑and‑drop or SQL‑like querying for power users.
* **RAG Chatbot:** Natural‑language Q\&A across the full corpus of reviews.

---

## 3. Key Business Insights (Illustrative)

### 3.1 Economy‑Class Passenger Trends
![Problem 1 Details](https://github.com/MarkPhamm/British-Airway/assets/99457952/665ff202-218a-4862-a130-98ce4c8584b9)
**Findings**

* Across airlines, ground‑staff service and boarding efficiency dominate complaints.
* Major international hubs (e.g. LHR, CDG, JFK) see the highest negative volume—often tied to long security queues and staff shortages.
* 92 % of low‑rating Economy reviews cite *at‑airport* factors rather than in‑flight experience.

![Problem 1](https://github.com/MarkPhamm/British-Airway/assets/99457952/fad27d46-f9c1-4187-94af-02da65d3f10b)
**Recommendations**

* Collaborate with ground‑handling partners to boost staffing during peak waves.
* Deploy self‑service kiosks and real‑time queue monitoring.

### 3.2 Premium‑Cabin Passenger Expectations


**Findings**

* Business & First passengers focus on seat comfort, bedding quality, and connectivity speed.
* Consistency gaps between aircraft sub‑fleets (older cabins vs. refurbished) drive dissatisfaction.
* Food quality is the second‑largest driver of 4‑star‑and‑below ratings.

**Recommendations**

* Accelerate fleet‑wide seat upgrade programmes.
* Introduce chef‑curated rotating menus with regional options.
* Guarantee minimum bandwidth per passenger on Wi‑Fi plans.

---

## 4. Next Steps

1. **Expand Data Sources** – Integrate on‑time‑performance and DOT complaint data for richer modelling.
2. **Real‑Time Ingestion** – Move to CDC‑style pipelines to surface insights within hours of review publication.
3. **Predictive Modelling** – Use sentiment plus operational variables to forecast future NPS movement by airline and route.
4. **Monetisation** – Offer benchmarking dashboards to airlines and airports via subscription.

---

*© 2025 Skytrax Global Airlines Analytics Project*



## 2. Architecture Overview




