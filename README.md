# Olist Customer Segmentation & Acquisition Quality

Behavioural segmentation of 94,721 marketplace customers, built end to end in
BigQuery and Power BI.

**Stack:** BigQuery (SQL), BigQuery ML (K-Means), Power BI, DAX, Power Query M

---

## The finding that reframed the project

The dataset has two customer identifiers. `customer_id` is issued fresh for every
order; `customer_unique_id` is the only field that resolves to a person. On the
wrong key the base looks like 99,441 one-time buyers.

On the correct key:

> **3.12% of customers ever place a second order** — 2,997 of 96,096.

That breaks textbook RFM. Frequency is 1 for 96.9% of the base, so clustering on
Recency, Frequency and Monetary would silently reduce to two informative
dimensions and produce segments describing noise on the frequency axis.

So this is not a loyalty segmentation. It is an **acquisition quality** question:
which customers does this marketplace acquire, and what separates them.

---

## Headline result

Four segments, on 94,721 customers and R$13.47M of segmented revenue:

| Segment | Customers | % of base | Revenue | % of revenue | Avg order value | Avg review |
|---|---|---|---|---|---|---|
| High-Value One-Time | 31,357 | 33.1% | R$8,556,012 | 62.9% | R$272.86 | 4.57 |
| Low-Value One-Time | 44,866 | 47.4% | R$2,343,229 | 17.2% | R$52.23 | 4.65 |
| Dissatisfied | 15,594 | 16.5% | R$1,813,880 | 13.4% | R$116.32 | 1.58 |
| Repeat Buyers | 2,904 | 3.1% | R$759,910 | 5.6% | R$123.74 | 4.14 |

**Two things worth acting on:**

1. **Revenue is inverted against customer count.** A third of customers produce
   nearly two thirds of revenue, while the largest group — almost half the base —
   produces a sixth. Undifferentiated acquisition spend is buying mostly the
   cheap segment.

2. **Dissatisfaction is concentrated among mid-value customers, not bargain
   hunters.** The Dissatisfied segment averages R$116.32 per order, more than
   double the Low-Value segment's R$52.23, at an average review of 1.58. That is
   R$1.8M of revenue attached to customers having a bad experience — an
   operations problem, not a marketing one.

---

## How the cluster count was chosen

Seven models were trained (k = 2 through 8) and compared on two measures.
Mean squared distance always falls as k rises, so the elbow matters, not the
minimum.

| k | Davies-Bouldin | Mean squared distance | Improvement |
|---|---|---|---|
| 2 | 1.910 | 4.196 | — |
| 3 | 1.510 | 3.166 | 1.030 |
| **4** | **1.130** | **2.379** | **0.787** |
| 5 | 1.185 | 2.058 | 0.321 |
| 6 | 1.181 | 1.786 | 0.272 |
| 7 | 1.084 | 1.522 | 0.265 |
| 8 | 1.200 | 1.455 | 0.067 |

The elbow is unambiguous at **k = 4**: improvement drops from 0.787 to 0.321 at
the next step. Davies-Bouldin reaches its global minimum at k = 7 (1.084), but
k = 4 is within 0.05 of it and is a clear local minimum, with DB rising at both
k = 5 and k = 6. Four segments are also actionable where seven would not be.

---

## Method

**Features.** Clustering used five numeric behavioural features per customer:
log total spend, log average order value, order count, average review score, and
days since last order (recency anchored to 2018-10-17, the last purchase date in
the dataset). State and product category were deliberately **excluded from the
model** and used only to profile the resulting segments — one-hot encoding 27
states and 70 categories would have swamped five behavioural features and turned
the segmentation into a geography exercise.

**Log transform.** Spend is heavily right-skewed: median R$89.90, mean R$143.07,
99th percentile R$1,009, maximum R$13,440. Untransformed, a handful of very high
spenders pull cluster centres toward themselves and compress the majority of
customers into one undifferentiated group. Logs preserve the ordering while
compressing the tail.

**No demographics.** The dataset contains no age, gender or income fields. This
is necessarily a behavioural segmentation, not a demographic one.

---

## Data model

Star schema in Power BI:

- `fact_order_items` — 112,650 rows, one per product line, carrying price,
  freight and order date
- `dashboard_data` — customer dimension, one row per customer, with segment
  assignment
- `dim_date` — date dimension, marked as the report's date table

Revenue measures are written against the fact table rather than the customer
table, so they split correctly by date. Aggregating spend from the customer
dimension would produce a figure that cannot be sliced by time.

---

## Repository contents

| File | What it is |
|---|---|
| `01_requirements.md` | Business requirements document — objectives, scope, 13 functional requirements, 5 user stories with acceptance criteria |
| `02_data_quality_log.md` | Every data quality issue found, its cause, and the decision taken |
| `sql/01_profiling.sql` | Table grain and quality checks |
| `sql/02_customer_features.sql` | Customer feature table |
| `sql/03_model_input.sql` | Log transform and model input view |
| `sql/04_kmeans_models.sql` | Seven K-Means models and the comparison |
| `sql/05_segment_profile.sql` | Segment and revenue profiling |
| `sql/06_dashboard_data.sql` | Export table for Power BI |
| `screenshots/` | Dashboard pages |

---

## Known limitations

- **Static historic data.** Orders run September 2016 to October 2018. There is
  no scheduled refresh; recency is measured from a fixed anchor date, not today.
- **State is a shipping attribute.** `customer_state` records the delivery
  address, not residence. 38 customers had orders delivered to more than one
  state and were resolved to a single value.
- **Review scores carry no date** in the customer table, so they cannot be
  trended over time without joining reviews into the fact layer.
- **1,375 customers excluded** from the model: 699 with no review on any order
  (NULL cannot be clustered, and imputing a score would invent behaviour) and
  676 whose orders have no item lines, consistent with cancelled or unfulfilled
  orders. Their orders remain in the fact table, which is why total product
  revenue (R$13.59M) slightly exceeds segmented revenue (R$13.47M).
- **Repeat rate is reported two ways.** 3.12% across all 96,096 customers in the
  dataset; 3.07% across the 94,721 modelled customers. Both are correct for
  their denominator.

---

## Data source

Brazilian E-Commerce Public Dataset by Olist, available on Kaggle.
Licensed CC BY-NC-SA 4.0. This analysis is non-commercial and provided with
attribution to Olist.
