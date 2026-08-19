# Data Quality Log
## Customer Segmentation & Acquisition Quality Dashboard

| Field | Value |
|---|---|
| Document ID | DQL-001 |
| Version | 1.0 |
| Author | Simarjit Singh Tuli |
| Traces to | BRD-001, FR-013 |
| Status | Open — stage 1 complete |

Every issue found while profiling and modelling the Olist dataset, with the
decision taken and its reason. Written as the work happened, not reconstructed
afterwards.

---

### DQ-001 — Reviews file failed to load

**Found.** The `olist_order_reviews_dataset.csv` import into BigQuery aborted
with "CSV table encountered too many errors" at row 1863, 100 errors reached.

**Cause.** The file contains free-text customer review comments in Portuguese.
Some of those comments contain line breaks inside the quoted field, so a single
logical review spans several physical lines. The default CSV parser treats every
physical line as a row and fails on the fragments.

**Decision.** Enabled *Allow quoted newlines* and *Allow jagged rows* on import.
All 99,224 rows loaded with none dropped, verified against the published row
count. The same options were applied to the remaining files as a precaution.

---

### DQ-002 — Order items are not one row per order

**Found.** `order_items` holds 112,650 rows against 99,441 orders. Distinct
`order_id` within the table is 98,666.

**Cause.** One row represents one product line within an order, not one order.
Roughly 13,000 orders contain more than one item. There is no quantity column;
each additional unit is its own row carrying its own price.

**Decision.** Revenue is summed at item level: `SUM(price)` returns
R$13,591,643.70 in product revenue plus R$2,251,909.54 in freight. Any value
that is true once per order (order count, order date, review score) must use
`COUNT(DISTINCT order_id)` or an equivalent guard, because joining through this
table repeats those values once per item line.

---

### DQ-003 — Monetary columns loaded as floating point

**Found.** Schema auto-detect typed `price` and `freight_value` as FLOAT64.
Aggregates returned values such as `13591643.700000547` and, at customer level,
`31.79999999999` and `184.8599999999`.

**Cause.** Binary floating point cannot represent most decimal fractions
exactly, so the error accumulates across large sums. This is a display and
reconciliation risk in a dashboard reporting currency: two visuals aggregating
the same underlying figures can disagree at the cent level.

**Decision.** All monetary outputs in `customer_features` are wrapped in
`ROUND(..., 2)` at the point of aggregation. Verified clean in the saved table.

---

### DQ-004 — One order has no payment record

**Found.** `order_payments` contains 103,886 rows covering 99,440 distinct
orders. The `orders` table holds 99,441. One order has no matching payment row.

**Cause.** Not investigated. Also noted that payments are not one row per order:
a single order may be settled across several instalments or vouchers, sequenced
by `payment_sequential`.

**Decision.** Logged and left alone. One record in 99,441 cannot move a segment
boundary, and the cost of investigating exceeds the value. Recorded here so the
gap is a known quantity rather than a silent one.

---

### DQ-005 — Customer identity is split across two keys

**Found.** The `customers` table has 99,441 rows, 99,441 distinct `customer_id`
values, and 96,096 distinct `customer_unique_id` values.

**Cause.** Olist issues a fresh `customer_id` for every order placed.
`customer_unique_id` is the only field that resolves back to a single person.
One individual in this dataset holds 17 separate `customer_id` values.

**Impact.** This is the single most consequential finding in the project.
Aggregating on `customer_id` makes every customer appear to be a one-time
buyer, which sets order frequency to a constant. A constant carries no
information, so a K-Means model built on Recency, Frequency and Monetary would
silently be clustering on two dimensions while reporting three, and the
resulting segments would describe noise on the frequency axis.

**Decision.** `customer_unique_id` is the customer key throughout. Grouping on
it gives the true repeat rate:

> **3.12% of customers ever place a second order — 2,997 of 96,096.**

This reframes the brief. With frequency near-constant across 96.9% of the base,
this is not a loyalty segmentation. It is an acquisition quality question.

---

### DQ-006 — Customers with orders delivered to multiple states

**Found.** Adding `customer_state` to the feature table raised the row count
from 95,420 to 95,458, breaking the one-row-per-customer grain for 38 customers.

**Cause.** `customer_state` records the delivery address, not the buyer's
residence. A customer who ships to two different states appears once per state.
By definition these are all repeat buyers, since a second state requires a
second order. Plausible explanations include gifts, relocation, or shipping to a
workplace.

**Decision.** Resolved with `ANY_VALUE(customer_state)` and grouping on
`customer_unique_id` alone, restoring 95,420 rows. At 38 of 95,420 the choice
between an arbitrary state and a modal state cannot affect any result. Noted
explicitly that state is a shipping attribute, so it is not read as evidence of
where a customer lives.

---

### DQ-007 — Customers with no review on any order

**Found.** 699 customers (0.73%) have a NULL `avg_review` in the feature table.

**Cause.** Reviews are optional. `order_reviews` covers 99,224 of 99,441 orders,
and these 699 customers placed no order that was ever reviewed. The LEFT JOIN
correctly preserves them; `AVG` over zero rows returns NULL.

**Decision.** NULL is retained rather than filled with zero, per BRD-001 A-3 and
R-3: "no review given" and "reviewed as one star" are different facts and must
not be collapsed. K-Means cannot accept a NULL input, so these 699 customers are
excluded from the model and the exclusion is reported alongside the results.
Imputing a mean was rejected as inventing behaviour for 0.73% of the base to
avoid a footnote.

---

### DQ-008 — Customers with orders but no item lines

**Found.** The feature table returns 95,420 customers against 96,096 in the
`customers` table. 676 customers are absent.

**Cause.** These customers' orders have no rows in `order_items`, so `SUM(price)`
returns NULL and the rows drop out of aggregation. Consistent with orders that
were cancelled or never fulfilled — the `orders` table shows 625 cancelled and
609 unavailable orders.

**Decision.** Excluded. A customer with no purchased item has no purchasing
behaviour to cluster on. Reported as a known exclusion rather than a gap.

---

### Open item — distribution of monetary features

Spend is heavily right-skewed:

| Statistic | Value (R$) |
|---|---|
| Minimum | 0.85 |
| Median | 89.90 |
| Mean | 143.07 |
| 99th percentile | 1,009.00 |
| Maximum | 13,440.00 |

The mean sits roughly 60% above the median and the maximum is over 13 times the
99th percentile. Because K-Means assigns clusters by distance, untransformed
values of this shape let a small number of very high spenders pull cluster
centres toward themselves, compressing the majority of customers into a single
undifferentiated group.

**Planned action.** Apply a log transform to `total_spend` and `avg_order_value`
before clustering, per BRD-001 FR-004. The figures above are the justification.
