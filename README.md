# cx_analytics
A robust set of sythetic datasets from the style of Amazon CXBT team in order to help you understand, practice and learn common VOC foundational concepts and principles.
<img width="1600" height="900" alt="amazon customer journey" src="https://github.com/user-attachments/assets/e8f11bb9-a25a-4feb-99b5-0867e32693ee" />

## CXBT (Customer Experience & Business Trends) — Project Focus

* **What CXBT is:** a cross-company, data-driven org focused on improving customer experience by turning signals into action. BIE/DE roles partner with PM/Science to measure, diagnose, and optimize CX with experiments, causal methods, and ML.
* **How access works:** think **federated funnel**—deep, domain-scoped entitlements (e.g., Discovery, Checkout, Delivery) plus curated cross-domain data products. Analysts own end-to-end outcomes with **least-privilege** data access.

### This repo focuses on Stages 1–3 of the customer journey

1. **Stage 1: Discover & Consider**
   Sessions, search, browse, PDP, help, reviews, and pre-purchase CES. KPIs: DPVR, ZRR, Help Bounce→Contact, CES, Review sentiment; **Predictive:** Propensity-to-Explore; **Prescriptive:** Query→Content recommendations.
   *UTC + local timestamps included; SQL-ready facts/dims.*

2. **Stage 2: Purchase & Checkout**
   Checkout sessions/steps, payments, orders, and purchase CES. KPIs: ATCR, Purchase Conversion Rate, Checkout Error Rate, Payment Decline Rate, CES (effort); **Predictive:** Purchase Propensity; **Prescriptive:** Promo/Flow recommender.
   *Calibrated conversion controls; marketplace/prime flags for slicing.*

3. **Stage 3: Fulfillment & Delivery**
   Promised vs actual delivery, carrier events, WISMO, post-delivery NPS. KPIs: OTD, Promise Accuracy, WISMO Contact Rate, Delivery Exception Rate, NPS; **Predictive:** Late-Risk score; **Prescriptive:** Promise Orchestrator.
   *UTC/local dual timestamps throughout; market + prime enrichment.*

**Implementation highlights:** normalized facts/dims, daily KPI tables by **marketplace** and **prime\_flag**, temporal rollups (day/week/month), and Postgres-friendly schemas for descriptive, diagnostic, predictive, and prescriptive analytics.


## Data Dictionary
Here’s a clean, copy-pasteable **data dictionary** for your GitHub README.
Types are suggested for Postgres. All `*_ts` columns are **TIMESTAMPTZ (UTC)** unless noted as “local”.

---

# Stage 1 — Discover & Consider

## kpi\_discover\_daily  *(by date × marketplace × prime\_flag)*

| Column                    | Type         | Description                     |
| ------------------------- | ------------ | ------------------------------- |
| dt                        | DATE         | UTC calendar date bucket.       |
| marketplace               | TEXT         | Market.                         |
| prime\_flag               | SMALLINT     | 1=Prime, 0=Non-Prime.           |
| list\_impressions         | BIGINT       | Sum PLP impressions.            |
| pdp\_views                | BIGINT       | Sum PDP views.                  |
| dpvr                      | NUMERIC(8,4) | PDP views / impressions.        |
| total\_searches           | BIGINT       | # searches.                     |
| zero\_results             | BIGINT       | # zero-result searches.         |
| zrr                       | NUMERIC(8,4) | zero\_results/total\_searches.  |
| help\_sessions            | BIGINT       | Help visits.                    |
| escalations               | BIGINT       | Help escalations.               |
| help\_bounce\_to\_contact | NUMERIC(8,4) | escalations/help\_sessions.     |
| ces\_responses            | BIGINT       | Pre-purchase CES count.         |
| ces\_avg                  | NUMERIC(6,2) | Mean CES.                       |
| ces\_top\_box\_rate       | NUMERIC(8,4) | Share ≤2.                       |
| sessions                  | BIGINT       | Session count in segment.       |
| explore\_propensity\_mean | NUMERIC(8,4) | Predictive explore score mean.  |
| explore\_high\_rate       | NUMERIC(8,4) | Share of sessions ≥0.60.        |
| reco\_insert\_module      | BOOLEAN      | Prescriptive trigger.           |
| reco\_module\_type        | TEXT         | Module type to insert.          |
| dominant\_query\_category | TEXT         | Daily dominant search category. |

---

# Stage 2 — Purchase & Checkout

## kpi\_purchase\_daily  *(by date × marketplace × prime\_flag)*

| Column                   | Type         | Description                 |
| ------------------------ | ------------ | --------------------------- |
| dt                       | DATE         | UTC date.                   |
| marketplace              | TEXT         | Market.                     |
| prime\_flag              | SMALLINT     | 1/0.                        |
| pdp\_views               | BIGINT       | PDP views.                  |
| atc                      | BIGINT       | Add-to-carts.               |
| atcr                     | NUMERIC(8,4) | ATC / PDP views.            |
| pdp\_sess                | BIGINT       | PDP sessions (unique).      |
| orders                   | BIGINT       | Orders placed.              |
| pcr                      | NUMERIC(8,4) | Orders / PDP sessions.      |
| co\_attempts             | BIGINT       | Checkout step attempts.     |
| co\_errors               | BIGINT       | Checkout errors.            |
| co\_err\_rate            | NUMERIC(8,4) | Errors / attempts.          |
| payments                 | BIGINT       | Payments attempted.         |
| declines                 | BIGINT       | Declined payments.          |
| pay\_decl\_rate          | NUMERIC(8,4) | Declines / payments.        |
| pm\_card\_share          | NUMERIC(8,4) | Card share.                 |
| pm\_wallet\_share        | NUMERIC(8,4) | Wallet share.               |
| pm\_cod\_share           | NUMERIC(8,4) | COD share.                  |
| pm\_giftcard\_share      | NUMERIC(8,4) | Gift card share.            |
| ces\_buy\_responses      | BIGINT       | Post-purchase CES count.    |
| ces\_buy\_avg            | NUMERIC(6,2) | Avg CES after purchase.     |
| ces\_buy\_topbox         | NUMERIC(8,4) | ≤2 share.                   |
| prop\_buy\_mean          | NUMERIC(8,4) | Mean propensity of buyers.  |
| prop\_buy\_high\_rate    | NUMERIC(8,4) | Share ≥0.60 among buyers.   |
| nb\_promo                | TEXT         | Next-best promo rec.        |
| nb\_flow                 | TEXT         | Next-best flow rec.         |
| promo\_applied\_rate     | NUMERIC(8,4) | Orders with promo / orders. |
| promo\_free\_ship\_share | NUMERIC(8,4) | Free shipping share.        |
| promo\_10pct\_share      | NUMERIC(8,4) | 10% off share.              |
| promo\_bundle\_share     | NUMERIC(8,4) | Bundle share.               |

*(If you built `fact_purchase_enriched`, list its added columns similarly—e.g., marketplace/prime\_flag attached to order rows.)*

---

# Stage 3 — Fulfillment & Delivery

## kpi\_fulfillment\_daily  *(by date × marketplace × prime\_flag)*

| Column                                                       | Type         | Description                                             |                |           |
| ------------------------------------------------------------ | ------------ | ------------------------------------------------------- | -------------- | --------- |
| dt                                                           | DATE         | **Delivery** date (UTC) unless noted.                   |                |           |
| marketplace                                                  | TEXT         | Market.                                                 |                |           |
| prime\_flag                                                  | SMALLINT     | 1/0.                                                    |                |           |
| orders                                                       | BIGINT       | Delivered orders that day.                              |                |           |
| otd\_rate                                                    | NUMERIC(8,4) | On-time share.                                          |                |           |
| promise\_accuracy\_mae\_hours                                | NUMERIC(8,2) | Mean                                                    | actual-promise | in hours. |
| exceptions                                                   | BIGINT       | Orders with exception.                                  |                |           |
| delivery\_exception\_rate                                    | NUMERIC(8,4) | exceptions/orders.                                      |                |           |
| wismo\_contacts                                              | BIGINT       | WISMO contacts attributed to delivered orders that day. |                |           |
| wismo\_contact\_rate                                         | NUMERIC(8,4) | wismo\_contacts/orders.                                 |                |           |
| nps\_responses                                               | BIGINT       | Post-delivery NPS responses.                            |                |           |
| nps\_score                                                   | NUMERIC(6,2) | NPS = %Promoters − %Detractors.                         |                |           |
| late\_risk\_mean                                             | NUMERIC(8,4) | Avg late risk (label day rollup aligned to dt).         |                |           |
| late\_risk\_high\_rate                                       | NUMERIC(8,4) | Share of orders with risk ≥0.7.                         |                |           |
| (optional) no\_change / proactive\_comms / expedite\_upgrade | NUMERIC(8,4) | Orchestrator action share (if computed).                |                |           |

---

> **Timezone note:** All `*_ts` are stored as **UTC**. “`*_local_ts`” columns are localized view columns—still TIMESTAMPTZ, but include the market’s offset (derived from `marketplace → tz` map).
