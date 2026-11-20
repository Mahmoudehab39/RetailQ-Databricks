# 🛒 RetailQ — Retail Data Engineering

### Medallion Lakehouse on Databricks Free Edition

![Databricks](https://img.shields.io/badge/Databricks-Free%20Edition-red)
![Catalog](https://img.shields.io/badge/Catalog-retail__q-blue)
![Architecture](https://img.shields.io/badge/Architecture-Bronze%20%2F%20Silver%20%2F%20Gold-green)
![Status](https://img.shields.io/badge/Job-Succeeded-brightgreen)

> **Implemented hands-on, step by step, and verified by row counts at every layer.**
> Not designed from scratch — see [Credit](#-credit--how-this-was-built) below.

---

## 📊 At a glance

| Layer | What | Verified count |
|---|---|---|
| 🥉 Bronze | Postgres product_catalog / inventory | **23 / 43** |
| 🥉 Bronze | Salesforce account / opportunity | **50 / 100** |
| 🥉 Bronze | Blob transactions (1,000 + 100 incremental) | **1,100** |
| 🥈 Silver | transactions / account / opportunity / product_catalog / inventory | **1,100 / 50 / 100 / 22 / 43** |
| 🥇 Gold | fact_sales / dim_customer / dim_product / fact_inventory / calendar | **1,100 / 50 / 22 / 43 / 121 days** |
| 📈 Dashboard | Total Revenue | **$302,224,791** |
| 📈 Dashboard | Transaction Count | **1,100** |

---

## 🏗️ Architecture

```text
Postgres (Neon) ─┐
Salesforce ──────┼──► Bronze ──► Silver ──► Gold ──► Metric View ──► Dashboard + Genie
Blob CSV ────────┘              (Lakeflow)   (Star Schema)              │
                                                                        ▼
                                                              Job: RetailQ_Orchestration
```

- **Sources:** Postgres (Neon), Salesforce, Blob CSV via Auto Loader
- **Silver:** Lakeflow Declarative Pipelines, with data-quality expectations (`expect` vs `expect_or_drop`)
- **Gold:** Star Schema — `fact_sales`, `dim_customer`, `dim_product`, `fact_inventory`, `calendar`
- **Semantic layer:** Metric View `retail_q.retail_semantic.retail_metrics`
- **Orchestration:** Databricks Job `RetailQ_Orchestration`

---

## 🙏 Credit & How This Was Built

This project follows **Narender Kumar's (DataBeli)** tutorial
**"Retail Data Engineering Project in Databricks Free Edition"**:

- 🎥 Video: https://www.youtube.com/watch?v=QHwszePV3GY
- 💻 Code: https://github.com/databeli/databricks_course/tree/main/17_RetailQ_Project

I implemented every step hands-on in Databricks Free Edition
(catalog `retail_q`, Medallion Bronze / Silver / Gold) and verified
each layer by row counts.

### What I did beyond the tutorial

- ✅ Set up the Neon Postgres source and logical replication myself
- ✅ Loaded the invalid rows one by one, so the data-quality rules had real rows to catch
- ✅ Fixed the Metric View calendar reference to `retail_q.retail_gold.calendar`
- ✅ Created the `retail_q.retail_semantic` schema
- ✅ Built my own dashboard (**"RetailQ – My Sales Dashboard"**) from scratch
- ✅ Created a Genie space (**"Retail Metrics Insight Agent"**) on the Metric View

### ⚠️ Known limitation — source data, not a code error

The Opportunity source file has **no Account column**, so `account_id`
is empty in all 1,100 transactions and **Unique Customers is 0**.
Any customer-based chart is therefore empty by design of the source data.

---

## 📈 My dashboard

**"RetailQ – My Sales Dashboard"** — 7 charts + 5 KPI counters, built
from scratch on the Metric View:

| KPI | Value |
|---|---|
| Total Revenue | **$302,224,791** |
| Transaction Count | **1,100** |
| Total Quantity Sold | **3,305** |
| Total Discount | **$4,946,843** |
| Average Transaction Value | **$274,749.81** |

**Charts:** Revenue by Product Category · Revenue Over Time · Revenue by
Product Brand · Total Quantity by Product Category · Revenue by Payment
Mode · Revenue by Sales Channel · Transaction Count by Opportunity Stage

**Genie — monthly Total Revenue:** Jan **$89.9M** · Feb **$86.0M** ·
Mar **$94.8M** (highest) · Apr **$30.7M** · May **$0.7M**
*(May is 1 May only.)*

---

## ⚙️ Job — RetailQ_Orchestration

5 tasks in order, all **Succeeded in 2m 48s**:

| # | Task | Time |
|---|---|---|
| 1 | bronze_to_silver | 31s |
| 2 | silver_to_gold | 51s |
| 3 | gold_views | 24s |
| 4 | calendar | 18s |
| 5 | dashboard_refresh | 41s |

**One real failure, fixed:** the first run failed with
`DELTA_SOURCE_TABLE_IGNORE_CHANGES` because of a MERGE in version 4
of `salesforce_bronze.opportunity`. It was fixed with a one-time
Full Refresh on `bronze_to_silver`, then the Full Refresh flag was
removed again.

---

## 🧠 What I learned most

- Data-quality rules: `expect` vs `expect_or_drop`
- Why a checkpoint proves an incremental load: **1,100 — not 2,100**
- How to read a null that is a source-data limit, not a bug
- Why a Streaming Table fails on a MERGE, and when a Full Refresh is the right fix
