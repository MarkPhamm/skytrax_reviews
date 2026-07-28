# Demo runbook — Skytrax Reviews Analytics Platform

Crib sheet for the live Show → Why → What-if walkthrough. One command (or path) per pillar.

## Spine

| Pillar | Repo | Open first |
| --- | --- | --- |
| EL | `skytrax_reviews_extract_load` | `README.md`, `dags/`, `terraform/` |
| Transform / DataOps | `skytrax_reviews_transformation` | `dbt/`, `.github/workflows/`, `docs/cicd.md` |
| Insight | `airline_customer_exp_analysis` · `frontier-reviews-dashboard` · `spirit_airlines_dashboard` | Mode PDFs under each repo; same `FCT_REVIEW` grain |
| Narrative | `skytrax_reviews` | [Platform walkthrough](https://markphamm.github.io/skytrax_reviews/) |

---

## Pillar 1 — Modeling

```bash
# ERD + medallion story
open skytrax_reviews_transformation/data_model/   # or assets in READMEs
# Grain: one row per review_id in fct_review; dims conformed
```

Live verify: walk `stg → int → dims + fct` in [dbt docs lineage](https://d38l3fc9bckvbz.cloudfront.net/#!/overview/ba_transformation?g_v=1).

---

## Pillar 2 — Transformation

```bash
cd skytrax_reviews_transformation/dbt
source ../dbt_venv/bin/activate   # or your venv
dbt debug --target dev
dbt run -s fct_review --target dev
# Second run should merge fewer/zero new rows (incremental watermark)
dbt run -s fct_review --target dev
```

Break-it-live (pre-staged branches on transformation repo):

```bash
git checkout demo/contract-break   # column type drift → contract fails / slim CI red
git checkout demo/bad-data         # rating=7 → range / accepted_values catch
```

---

## Pillar 3 — Governance

```bash
# Freshness (warn 3d / error 7d)
dbt source freshness --target prod

# Tests + failure quarantine tables (store_failures)
dbt test -s fct_review --target prod
# Inspect: select * from <db>.dbt_test__audit.<test_name> limit 20;

# Spirit fleet plausibility (warn — free-text aircraft noise)
dbt test -s assert_spirit_aircraft_airbus --target prod

# Masking A/B
# Snowflake: use role SKYTRAX_ANALYST  → hashed name / nationality
#            use role SKYTRAX_TRANSFORMER → clear text
# review_text stays readable for BI (not hashed)
```

EL quality + quarantine:

```bash
# Rejected files leave processed/ → quarantine/<type>/YYYY/MM/...
# LOAD_AUDIT reconciliation
# select * from skytrax_reviews_db.raw.load_audit order by loaded_at desc limit 10;
```

Elementary:

```bash
dbt run -s elementary
edr report   # publish path documented in docs/observability.md
```

Control matrix slide in the deck maps each control → tool → evidence → command.

---

## Pillar 4 — Insight

```bash
# Mode dashboards (PDF backups in each insight repo)
# Same metric grain — filter by airline:
# select airline,
#        count(*) as reviews,
#        avg(average_rating) as avg_rating,
#        avg(iff(recommended,1,0)) as pct_recommended
# from marts.fct_review_enriched
# where airline in ('Delta Air Lines', 'Frontier Airlines', 'Spirit Airlines')
# group by 1
# order by 1;

# Semantic layer
dbt parse
mf query --metrics avg_rating,pct_recommended --group-by rating_band
```

Review counts (refresh deck/READMEs if these drift):

```sql
select airline, count(*) as reviews
from marts.fct_review_enriched
where airline in ('Delta Air Lines', 'Frontier Airlines', 'Spirit Airlines')
group by 1
order by 1;
-- Expected ~2912 Delta · ~3533 Frontier · ~4698 Spirit (fct_review snapshot)
```

Distinctive levers to defend live:

| Airline | Insight | Action |
| --- | --- | --- |
| Delta | Economy vs Premium drivers diverge | Cabin-specific plays (Wi‑Fi + Economy value + Premium comfort) |
| Frontier | Lowest ULCC recommend rate vs peers | Close value gap — Economy entertainment + seat comfort |
| Spirit | Chronic dissatisfaction; Business worst | IFE/Wi‑Fi SLAs + rebuild Business + MIA/MEX/GOT ops |
---

## Pillar 5 — DataOps

```bash
# Slim CI story: pr_checks.yml → clone + state:modified+
# CD: deploy_main.yml
#   schedule → freshness + snapshot + full dbt build
#   push    → state:modified+ --defer --favor-state
# Docs: only index.html / manifest.json / catalog.json → CloudFront
open 'https://d38l3fc9bckvbz.cloudfront.net/#!/overview/ba_transformation?g_v=1'
```

Cost attribution:

```sql
select query_tag, count(*), sum(total_elapsed_time)
from snowflake.account_usage.query_history
where start_time > dateadd('day', -7, current_timestamp())
group by 1 order by 3 desc;
-- tags: local_dev | github_action_ci | github_action_deploy | airflow_cosmos
```

---

## Time-travel recovery (rehearse, no build)

```sql
-- Bad load? Inspect prior state, then zero-copy clone or swap.
create table marts.fct_review_recovered clone marts.fct_review at (offset => -3600);
```

---

## Don’t show live

- `.env`, `profiles.yml` passwords, AWS keys, Snowflake account locators
- Real Mode workspace URL if it embeds other customer data
