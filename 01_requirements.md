# Business Requirements Document
## Customer Segmentation & Acquisition Quality Dashboard

| Field | Value |
|---|---|
| Document ID | BRD-001 |
| Version | 1.1 |
| Author | Simarjit Singh Tuli |
| Date raised | 2026-08-10 |
| Status | Delivered — see Delivery notes |
| Data source | Olist Brazilian E-Commerce Public Dataset (CC BY-NC-SA 4.0) |

---

## 1. Background

Olist is a Brazilian marketplace connecting small sellers to major e-commerce
platforms. The dataset covers 99,441 orders placed between September 2016 and
October 2018 by 96,096 distinct customers across 27 states.

An initial data profile surfaced a finding that reframes the entire brief:

> **3.12% of customers ever place a second order.** 2,997 of 96,096.

This has a direct methodological consequence. Standard RFM segmentation assumes
frequency varies meaningfully across the customer base. Here it does not.
Frequency equals 1 for 96.9% of customers, so clustering on Recency, Frequency
and Monetary silently reduces to two informative dimensions, and the resulting
segments describe noise on the F axis.

The brief is therefore **not** a loyalty segmentation. It is an acquisition
quality question: which customers does this marketplace acquire, which of those
cohorts show any repeat signal, and what in the order experience predicts the
difference.

## 2. Problem statement

The business cannot currently distinguish between customer cohorts at
acquisition. Marketing spend is undifferentiated by segment, and there is no
view linking delivery experience, payment behaviour or category mix to whether a
customer returns. Without segmentation, retention effort cannot be targeted.

## 3. Objectives

| ID | Objective | Success criteria |
|---|---|---|
| OBJ-1 | Segment the customer base on behavioural and experiential features | Cluster count justified by quantitative method, not assumed |
| OBJ-2 | Profile each segment in business language | Every segment has a name, a size, a revenue share and one recommended action |
| OBJ-3 | Expose segments in a self-serve dashboard | Stakeholder can filter to a segment and reach customer-level detail unaided |
| OBJ-4 | Quantify the repeat-purchase problem | Repeat rate reported overall, by state and by category |

## 4. Scope

**In scope**
- Customer-level feature engineering from orders, items, payments, reviews
- Unsupervised segmentation with a justified cluster count
- Star schema model suitable for Power BI
- Dashboard: executive overview, segment profile, geographic view, drill-through
- Data quality documentation

**Out of scope**
- Predictive churn or lifetime value modelling (deferred to project two)
- Portuguese review text sentiment analysis
- Seller-side performance analysis
- Any real-time or scheduled refresh

## 5. Stakeholders

| Role | Party | Responsibility |
|---|---|---|
| Analyst / BA | Simarjit Singh Tuli | Requirements, build, test |
| Business stakeholder (proxy) | Marketplace growth team | Accepts or rejects against criteria |
| UAT reviewer | External reviewer | Independent acceptance pass |

Note: this is a self-initiated analysis. The stakeholder role is played as a
proxy for a marketplace growth function. UAT is performed by an independent
reviewer rather than the author.

## 6. Assumptions and constraints

| ID | Statement | Type |
|---|---|---|
| A-1 | `customer_unique_id` is the true customer key; `customer_id` is order-scoped | Assumption |
| A-2 | Orders not reaching `delivered` are excluded from behavioural features | Assumption |
| A-3 | Monetary value is item price plus freight, reconciled against payments | Assumption |
| C-1 | Dataset is static and historic; no live refresh | Constraint |
| C-2 | Delivery is 2 working days from requirements sign-off | Constraint |
| C-3 | Licence is non-commercial; attribution required on publication | Constraint |

## 7. Data sources

| Table | Rows | Role in model |
|---|---|---|
| olist_orders_dataset | 99,441 | Fact grain driver |
| olist_order_items_dataset | 112,650 | Fact table |
| olist_order_payments_dataset | 103,886 | Payment behaviour features |
| olist_order_reviews_dataset | 99,224 | Experience features |
| olist_customers_dataset | 99,441 | Customer dimension |
| olist_products_dataset | 32,951 | Product dimension |
| olist_sellers_dataset | 3,095 | Seller dimension |
| olist_geolocation_dataset | 1,000,163 | Geography, requires deduplication |
| product_category_name_translation | 72 | Category labels |

## 8. Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-001 | Resolve customer identity to `customer_unique_id` and report the collapse from 99,441 to 96,096 | Must |
| FR-002 | Exclude non-delivered orders from behavioural features and record the volume excluded | Must |
| FR-003 | Derive per-customer features: recency, frequency, monetary, average order value, average review score | Must |
| FR-004 | Apply distribution correction to skewed monetary features before clustering | Must |
| FR-005 | Determine cluster count using elbow and silhouette; record both | Must |
| FR-006 | Assign every customer to exactly one segment | Must |
| FR-007 | Produce a star schema with conformed dimensions and a date dimension | Must |
| FR-008 | Dashboard page: executive overview with customer count, revenue, repeat rate, segment mix | Must |
| FR-009 | Dashboard page: segment profile comparing feature means across segments | Must |
| FR-010 | Dashboard page: geographic view of segments and repeat rate by state | Should |
| FR-014 | Cluster on numeric behavioural features only; use state and category to profile segments after the fact | Must |
| FR-011 | Drill-through from any segment to customer-level detail | Should |
| FR-012 | Deduplicate geolocation to one coordinate pair per zip prefix | Dropped — see D-2 |
| FR-013 | Document every data quality decision in a written log | Must |

## 9. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-1 | Report page renders in under 3 seconds on desktop |
| NFR-2 | Model refreshes end to end in under 5 minutes |
| NFR-3 | All measures written in DAX, not calculated columns, where a measure is viable |
| NFR-4 | Transformation code is reproducible from raw CSVs with no manual steps |
| NFR-5 | Attribution to Olist present on any published artefact |

## 10. User stories

**US-01** — As a growth lead, I want to see what share of customers ever return,
so that I can size the retention problem.
- Given the overview page, when it loads, then overall repeat rate is displayed against total customers
- Given the overview page, when I filter to a state, then repeat rate recalculates for that state

**US-02** — As a growth lead, I want customers grouped into behavioural segments,
so that I can target spend rather than treat the base as uniform.
- Given the segment page, when it loads, then every segment shows name, customer count, revenue share and average order value
- Given the segmentation, when I check totals, then segment customer counts sum to the full delivered-order customer base

**US-03** — As a growth lead, I want to know whether delivery experience differs
by segment, so that I can tell an operations problem from a marketing one.
- Given the segment page, when I compare segments, then mean delivery delay and mean review score are shown per segment

**US-04** — As a growth lead, I want to reach the customers behind a segment,
so that I can validate the profile against real records.
- Given any segment visual, when I drill through, then a customer-level table opens filtered to that segment

**US-05** — As an analyst, I want the cluster count to be justified, so that the
segmentation withstands challenge.
- Given the documentation, when the method is reviewed, then elbow and silhouette outputs are shown with the chosen k stated and reasoned

## 11. Risks and open questions

| ID | Item | Response |
|---|---|---|
| R-1 | Low repeat rate may produce weakly separated clusters | Report silhouette honestly; a weak result reported plainly is a valid finding |
| R-2 | Geolocation duplicates could inflate joins | Deduplicate to one row per zip prefix before joining |
| R-3 | Review coverage is incomplete relative to orders | Treat missing review score as null, never as zero |
| Q-1 | Should cancelled orders count toward monetary value? | **Closed: no.** Orders with no item lines produce no revenue and are excluded (676 customers) |
| Q-2 | Is a state-level or city-level geography grain required? | **Closed: state.** 4,119 cities across 96,096 customers averages ~23 per city, too granular to segment on |

## 12. Delivery notes

Changes made between draft and delivery, with reasons:

| ID | Change | Reason |
|---|---|---|
| D-1 | FR-003 reduced to five features. Delivery delay, freight share, payment instalments and dominant category were dropped from the model | Delivery delay and freight share were not required to separate the segments that emerged. Dominant category requires a per-customer ranking step and is categorical, so it could not be a K-Means input. Payment instalments was parked as lower value than the effort |
| D-2 | FR-012 dropped | Customer state is available directly on the customer table. Geolocation coordinates were only needed for a map, and the map was replaced by a sorted bar chart (see D-3), so the 1M-row deduplication was never on the critical path |
| D-3 | Map visual replaced with a sorted bar chart on FR-010 | Power BI's map plotted Brazilian two-letter state codes onto US states (MT to Montana, PA to Pennsylvania). With 27 states a sorted bar chart is both correct and easier to read |
| D-4 | FR-014 added | Discovered during feature selection: one-hot encoding 27 states and 70 categories would have contributed ~97 columns against 5 behavioural features, making the segmentation a geography exercise. Categorical fields profile the segments instead |
| D-5 | Cluster count set to 4 | Seven models trained (k=2 to 8). Elbow in mean squared distance falls at k=4; Davies-Bouldin at k=4 is 1.130 against a global minimum of 1.084 at k=7. Four segments are actionable where seven would not be. Satisfies FR-005 |

Requirements delivered: FR-001 to FR-011, FR-013, FR-014.
Requirements dropped with reason: FR-012.

## 13. Sign-off

| Role | Name | Date | Decision |
|---|---|---|---|
| Business stakeholder | | | |
| UAT reviewer | | | |
