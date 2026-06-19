# fabric-medallion-insurance-analytics
# Medallion Analytics Pipeline on Microsoft Fabric — Insurance Charges

An end-to-end data analytics pipeline built on **Microsoft Fabric**, taking raw data
from source all the way to an interactive Power BI dashboard using the **medallion
architecture** (Bronze → Silver → Gold) and **Kimball-style dimensional modeling**.

> **Note:** This is a learning project, built to master the Microsoft Fabric platform
> end-to-end on a clean, simple dataset before applying the same architecture to a
> larger healthcare analytics project on Medicare / FHIR claims data. The *plumbing*
> here is production-shaped; the dataset is deliberately simple so the focus stays on
> the platform and patterns.

---

## Business Context

The dataset contains personal medical insurance records (age, sex, BMI, region,
smoking status, dependents, and the charges billed). The goal is to answer questions
a health insurer actually cares about:

- What drives insurance cost the most?
- How do smokers compare to non-smokers?
- Does cost vary by region or BMI category?
- What is the average charge per member, and how does it shift across segments?

**Headline finding:** smokers are billed roughly **4× more** on average than
non-smokers (~$32,000 vs ~$8,400), making smoking status the single largest cost driver,
with region a meaningful secondary factor (Southeast highest).

---

## Architecture

```
                 MEDALLION ARCHITECTURE ON MICROSOFT FABRIC
==================================================================================

  SOURCE              BRONZE                SILVER                 GOLD
 ┌────────┐         ┌──────────┐         ┌────────────┐        ┌──────────────────┐
 │  Raw   │  ingest │ raw copy │  clean  │  cleaned + │ model  │  star schema     │
 │  CSV   │ ──────▶ │ (as-is,  │ ──────▶ │  enriched  │ ─────▶ │  fact + 4 dims   │
 │ (web)  │         │ untouched)│        │  data      │        │                  │
 └────────┘         └──────────┘         └────────────┘        └──────────────────┘
                    bronze_insurance      silver_insurance       fact_charges
                                          + is_smoker            dim_region
                                          + bmi_category         dim_smoker
                                          + age_band             dim_bmi
                                          + rounded charges      dim_demographics
                                                                        │
                                                                        │ Direct Lake
                                                                        ▼
                                                              ┌──────────────────┐
                                                              │   POWER BI       │
                                                              │ semantic model   │
                                                              │ + DAX measures   │
                                                              │ + report         │
                                                              └──────────────────┘

 LAYER PURPOSE
 • Bronze  — faithful, immutable copy of the source (replay-able truth)
 • Silver  — data quality: cleaning, type fixes, derived columns
 • Gold    — shape for consumption: facts (measures) + dimensions (labels)
==================================================================================
```

---

## The Star Schema (Gold)

```
              dim_demographics
                     │
   dim_region ── fact_charges ── dim_smoker
                     │
                 dim_bmi
```

- **Fact:** `fact_charges` — the measurable numbers (`charges`, `children`) plus
  surrogate foreign keys to each dimension.
- **Dimensions:** `dim_region`, `dim_smoker`, `dim_bmi`, `dim_demographics` — the
  descriptive labels we slice by, each with an integer **surrogate key**.
- Relationships are **one-to-many** (dimension → fact), with single-direction
  cross-filtering — the standard, performant star pattern.

Surrogate keys (small integers generated in Gold) keep the fact table narrow and make
joins fast — the foundation of a responsive Power BI model.

---

## Tech Stack

| Layer            | Technology                                    |
|------------------|-----------------------------------------------|
| Storage          | OneLake (Delta tables)                         |
| Ingestion        | Fabric Notebook (Python) — fetch & land        |
| Transformation   | Fabric Notebook (PySpark) — clean, enrich, model |
| Table format     | Delta Lake                                     |
| Modeling         | Power BI semantic model (Direct Lake)          |
| Metrics          | DAX measures                                    |
| Visualization    | Power BI report                                |

---

## Repository Contents

| File                       | Layer  | What it does                                            |
|----------------------------|--------|---------------------------------------------------------|
| `nb_insurance_bronze.ipynb`| Bronze | Ingests raw CSV from the web, lands it untouched as a Delta table; includes EDA |
| `nb_insurance_silver.ipynb`| Silver | Reads Bronze, cleans/standardizes, adds `is_smoker`, `bmi_category`, `age_band` |
| `nb_insurance_gold.ipynb`  | Gold   | Builds the star schema: fact table + 4 dimensions with surrogate keys |

---

## DAX Measures (in the semantic model)

```dax
Total Charges  = SUM(fact_charges[charges])
Average Charge = AVERAGE(fact_charges[charges])
Total Members  = COUNTROWS(fact_charges)
```

These are defined once on the semantic model so every report inherits a single,
consistent source of truth for each metric.

---

## Key Concepts Demonstrated

- Medallion architecture (Bronze / Silver / Gold) and *why each layer exists*
- Reading from and writing to a Fabric Lakehouse with Spark (avoiding cross-engine
  path pitfalls — Spark writes, Spark reads)
- PySpark transformations: `withColumn`, type casting, chained `when().otherwise()`
  for ordered bucketing
- Kimball dimensional modeling: facts vs dimensions, grain, surrogate keys
- Power BI semantic modeling with Direct Lake, one-to-many relationships, and DAX

---

## Next Steps

Apply this same architecture to a larger, production-style dataset:
**Medicare claims analytics on synthetic FHIR data (Synthea / CMS Blue Button 2.0)** —
where the added challenge is shredding nested FHIR JSON bundles into flat Silver tables,
on top of the same Bronze → Silver → Gold → Power BI foundation built here.

---

## Data Source

Medical Cost Personal Dataset (publicly available, commonly used for analytics practice).
