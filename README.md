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

## dim\_customer

| Column        | Type              | Description                                                        |
| ------------- | ----------------- | ------------------------------------------------------------------ |
| customer\_id  | TEXT (PK)         | Customer key (A–Z/0–9).                                            |
| marketplace   | TEXT              | ISO-like market code (US, UK, DE, JP, IN, CA, FR, ES, IT, AU, MX). |
| prime\_member | BOOLEAN           | Whether the account is Prime.                                      |
| device\_pref  | TEXT              | Customer’s device preference (mobile/desktop/tablet).              |
| traffic\_bias | TEXT              | Typical traffic source (organic/paid\_ads/direct/social/email).    |
| created\_at   | TIMESTAMPTZ (UTC) | Customer record creation time.                                     |

## dim\_product

| Column   | Type          | Description              |
| -------- | ------------- | ------------------------ |
| asin     | TEXT (PK)     | Product ASIN (10 chars). |
| category | TEXT          | Product category.        |
| brand    | TEXT          | Brand cluster.           |
| price    | NUMERIC(12,2) | Current list price.      |

## fact\_session

| Column             | Type                      | Description                                                       |
| ------------------ | ------------------------- | ----------------------------------------------------------------- |
| session\_id        | TEXT (PK)                 | Session key.                                                      |
| customer\_id       | TEXT (FK → dim\_customer) | Visitor customer id (can be null for anon if you add that later). |
| marketplace        | TEXT                      | Market for the session.                                           |
| prime\_member      | BOOLEAN                   | Prime status as of session.                                       |
| device             | TEXT                      | Device in session.                                                |
| traffic\_source    | TEXT                      | Session traffic source.                                           |
| session\_start\_ts | TIMESTAMPTZ (UTC)         | Session start time.                                               |

## fact\_search

| Column          | Type                      | Description                  |
| --------------- | ------------------------- | ---------------------------- |
| session\_id     | TEXT (FK → fact\_session) | Session.                     |
| query\_category | TEXT                      | Broad category of the query. |
| query           | TEXT                      | Query string token.          |
| results\_count  | INTEGER                   | Search results returned.     |
| zero\_result    | BOOLEAN                   | True if 0 results.           |
| search\_ts      | TIMESTAMPTZ (UTC)         | Timestamp of search.         |

## fact\_browse

| Column            | Type                     | Description            |
| ----------------- | ------------------------ | ---------------------- |
| session\_id       | TEXT (FK)                | Session.               |
| marketplace       | TEXT                     | Market for event.      |
| asin              | TEXT (FK → dim\_product) | Product shown/clicked. |
| list\_impressions | INTEGER                  | # impressions (PLP).   |
| list\_clicks      | INTEGER                  | # clicks from PLP.     |
| browse\_ts        | TIMESTAMPTZ (UTC)        | Event time.            |

## fact\_pdp

| Column         | Type                     | Description                  |
| -------------- | ------------------------ | ---------------------------- |
| session\_id    | TEXT (FK)                | Session.                     |
| asin           | TEXT (FK → dim\_product) | Product viewed.              |
| pdp\_views     | INTEGER                  | PDP views in session window. |
| add\_to\_carts | INTEGER                  | Add-to-cart count.           |
| pdp\_ts        | TIMESTAMPTZ (UTC)        | PDP event anchor time.       |

## fact\_help

| Column                 | Type              | Description                      |
| ---------------------- | ----------------- | -------------------------------- |
| session\_id            | TEXT (FK)         | Session.                         |
| topic                  | TEXT              | Help topic (shipping/returns/…). |
| help\_ts               | TIMESTAMPTZ (UTC) | Help page time.                  |
| escalated\_to\_contact | BOOLEAN           | Escalated to agent/contact.      |

## fact\_review\_interaction

| Column             | Type         | Description                      |
| ------------------ | ------------ | -------------------------------- |
| session\_id        | TEXT (FK)    | Session.                         |
| asin               | TEXT (FK)    | Product whose reviews were read. |
| reviews\_seen      | INTEGER      | Count of reviews read/scrolled.  |
| avg\_star\_exposed | NUMERIC(4,2) | Avg star rating exposed.         |
| sentiment\_exposed | NUMERIC(6,3) | Sentiment score (-1..+1).        |

## fact\_ces\_pre\_purchase

| Column        | Type              | Description               |
| ------------- | ----------------- | ------------------------- |
| session\_id   | TEXT (FK)         | Session.                  |
| ces\_score    | INTEGER           | 1..7 (lower=less effort). |
| ces\_top\_box | BOOLEAN           | True if ≤2.               |
| instrument    | TEXT              | “CES\_PRE\_PURCHASE”.     |
| created\_at   | TIMESTAMPTZ (UTC) | Survey time.              |

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

## conversion\_plan

| Column                       | Type         | Description                      |
| ---------------------------- | ------------ | -------------------------------- |
| customer\_id                 | TEXT (PK/FK) | Customer.                        |
| will\_convert\_anytime       | BOOLEAN      | Ground-truth plan flag.          |
| base\_conversion\_propensity | NUMERIC(6,4) | Baseline propensity (0..1).      |
| planned\_order\_id           | TEXT         | If converting, planned order id. |
| planned\_conversion\_days    | INTEGER      | Lead time days.                  |

---

# Stage 2 — Purchase & Checkout

## fact\_checkout\_session

| Column                     | Type                      | Description                        |
| -------------------------- | ------------------------- | ---------------------------------- |
| checkout\_session\_id      | TEXT (PK)                 | Checkout session id.               |
| session\_id                | TEXT (FK → fact\_session) | Source session.                    |
| customer\_id               | TEXT (FK → dim\_customer) | Customer.                          |
| checkout\_start\_ts        | TIMESTAMPTZ (UTC)         | Start of checkout.                 |
| checkout\_start\_local\_ts | TIMESTAMPTZ (local)       | Localized start.                   |
| abandoned                  | BOOLEAN                   | True if no purchase.               |
| abandon\_step              | TEXT                      | Last step before abandon (if any). |

## fact\_checkout\_step

| Column                 | Type                | Description                                        |
| ---------------------- | ------------------- | -------------------------------------------------- |
| checkout\_session\_id  | TEXT (FK)           | Checkout session.                                  |
| step\_name             | TEXT                | cart/address/shipping/payment/review/place\_order. |
| attempt                | SMALLINT            | Attempt # (1, 2…).                                 |
| error\_flag            | BOOLEAN             | Step error happened.                               |
| error\_code            | TEXT                | Error code (e.g., PAY\_DECLINED).                  |
| step\_start\_ts        | TIMESTAMPTZ (UTC)   | Start time.                                        |
| step\_end\_ts          | TIMESTAMPTZ (UTC)   | End time.                                          |
| step\_start\_local\_ts | TIMESTAMPTZ (local) | Local start.                                       |
| step\_end\_local\_ts   | TIMESTAMPTZ (local) | Local end.                                         |

## fact\_payment

| Column                | Type                    | Description                     |
| --------------------- | ----------------------- | ------------------------------- |
| order\_id             | TEXT (FK → fact\_order) | Order.                          |
| checkout\_session\_id | TEXT (FK)               | Checkout session.               |
| payment\_method       | TEXT                    | card/wallet/cod/giftcard.       |
| declined              | BOOLEAN                 | True if first attempt declined. |
| decline\_code         | TEXT                    | Decline reason.                 |
| payment\_ts           | TIMESTAMPTZ (UTC)       | Payment time (post-review).     |
| payment\_local\_ts    | TIMESTAMPTZ (local)     | Local payment time.             |

## fact\_order

| Column                | Type                     | Description                            |
| --------------------- | ------------------------ | -------------------------------------- |
| order\_id             | TEXT (PK)                | Order id.                              |
| checkout\_session\_id | TEXT (FK)                | Checkout session.                      |
| customer\_id          | TEXT (FK)                | Customer.                              |
| session\_id           | TEXT (FK)                | Source session.                        |
| asin                  | TEXT (FK → dim\_product) | Product purchased.                     |
| quantity              | INTEGER                  | Units.                                 |
| order\_amount         | NUMERIC(12,2)            | Net after promo.                       |
| currency              | TEXT                     | ISO currency.                          |
| order\_ts             | TIMESTAMPTZ (UTC)        | Order time.                            |
| order\_local\_ts      | TIMESTAMPTZ (local)      | Local order time.                      |
| promo\_applied        | BOOLEAN                  | Promo applied flag.                    |
| promo\_type           | TEXT                     | free\_shipping/10pct\_off/bundle/NULL. |

## fact\_ces\_purchase

| Column                 | Type                | Description               |
| ---------------------- | ------------------- | ------------------------- |
| order\_id              | TEXT (FK)           | Order.                    |
| ces\_score             | INTEGER             | 1..7 (lower=less effort). |
| ces\_top\_box          | BOOLEAN             | ≤2 flag.                  |
| created\_at            | TIMESTAMPTZ (UTC)   | Survey time.              |
| created\_at\_local\_ts | TIMESTAMPTZ (local) | Local survey time.        |

## session\_scored

| Column                 | Type              | Description                  |
| ---------------------- | ----------------- | ---------------------------- |
| session\_id            | TEXT (FK)         | Session.                     |
| customer\_id           | TEXT (FK)         | Customer.                    |
| prime\_member          | BOOLEAN           | Prime in session.            |
| device                 | TEXT              | Device.                      |
| traffic\_source        | TEXT              | Traffic source.              |
| session\_start\_ts     | TIMESTAMPTZ (UTC) | Session start.               |
| pdp\_views             | INTEGER           | PDP views (from Stage-1).    |
| add\_to\_carts         | INTEGER           | ATC (from Stage-1).          |
| zero\_result\_seen     | SMALLINT          | 1 if saw zero-result search. |
| help\_escalation       | SMALLINT          | 1 if help escalated.         |
| avg\_star\_seen        | NUMERIC(4,2)      | Avg star rating exposed.     |
| purchase\_propensity   | NUMERIC(6,4)      | Predictive score (0..1).     |
| promo\_recommendation  | TEXT              | Prescriptive promo.          |
| flow\_recommendation   | TEXT              | Prescriptive flow.           |
| recommendation\_reason | TEXT              | Why.                         |
| checkout\_session\_id  | TEXT (FK)         | Linked checkout id (if any). |
| checkout\_start\_ts    | TIMESTAMPTZ (UTC) | Checkout start (if any).     |

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

## fact\_fulfillment

| Column                         | Type                       | Description                          |
| ------------------------------ | -------------------------- | ------------------------------------ |
| order\_id                      | TEXT (PK/FK → fact\_order) | Order.                               |
| marketplace                    | TEXT                       | Market of order.                     |
| shipping\_method               | TEXT                       | standard/two\_day/next\_day.         |
| carrier                        | TEXT                       | AMZL/UPS/USPS/FedEx/DHL.             |
| label\_created\_ts             | TIMESTAMPTZ (UTC)          | Label creation.                      |
| handed\_to\_carrier\_ts        | TIMESTAMPTZ (UTC)          | Carrier pickup.                      |
| promised\_delivery\_ts         | TIMESTAMPTZ (UTC)          | Active promise (post-orchestration). |
| actual\_delivery\_ts           | TIMESTAMPTZ (UTC)          | Delivery confirmed.                  |
| exception\_flag                | BOOLEAN                    | Delivery exception occurred.         |
| exception\_code                | TEXT                       | WEATHER/ADDR\_ISSUE/…                |
| on\_time                       | BOOLEAN                    | Delivered ≤ promise.                 |
| promise\_error\_hours          | NUMERIC(8,2)               | Actual - Promise (hrs; <0 early).    |
| label\_created\_local\_ts      | TIMESTAMPTZ (local)        | Localized.                           |
| handed\_to\_carrier\_local\_ts | TIMESTAMPTZ (local)        | Localized.                           |
| promised\_delivery\_local\_ts  | TIMESTAMPTZ (local)        | Localized.                           |
| actual\_delivery\_local\_ts    | TIMESTAMPTZ (local)        | Localized.                           |

## fact\_carrier\_event

| Column           | Type                | Description                                                                         |
| ---------------- | ------------------- | ----------------------------------------------------------------------------------- |
| order\_id        | TEXT (FK)           | Order.                                                                              |
| event\_type      | TEXT                | label\_created/picked\_up/in\_transit\_scan/exception/out\_for\_delivery/delivered. |
| event\_ts        | TIMESTAMPTZ (UTC)   | Event time.                                                                         |
| event\_local\_ts | TIMESTAMPTZ (local) | Local event time.                                                                   |

## fact\_wismo\_contact

| Column             | Type                | Description         |
| ------------------ | ------------------- | ------------------- |
| order\_id          | TEXT (FK)           | Order.              |
| contact\_ts        | TIMESTAMPTZ (UTC)   | Contact time.       |
| contact\_local\_ts | TIMESTAMPTZ (local) | Local contact time. |
| contact\_channel   | TEXT                | phone/chat/email.   |
| escalated          | BOOLEAN             | Escalated contact.  |

## fact\_nps\_post\_delivery

| Column                 | Type                | Description        |
| ---------------------- | ------------------- | ------------------ |
| order\_id              | TEXT (FK)           | Order.             |
| nps\_score             | SMALLINT            | 0..10.             |
| promoter               | BOOLEAN             | 9–10.              |
| passive                | BOOLEAN             | 7–8.               |
| detractor              | BOOLEAN             | ≤6.                |
| created\_at            | TIMESTAMPTZ (UTC)   | Survey time.       |
| created\_at\_local\_ts | TIMESTAMPTZ (local) | Local survey time. |

## fact\_fulfillment\_enriched

| Column                               | Type              | Description                                    |
| ------------------------------------ | ----------------- | ---------------------------------------------- |
| (all columns from fact\_fulfillment) |                   |                                                |
| session\_id                          | TEXT (FK)         | Session of order.                              |
| customer\_id                         | TEXT (FK)         | Customer.                                      |
| prime\_flag                          | SMALLINT          | 1/0 at order time.                             |
| dt\_order                            | DATE              | Order date (UTC).                              |
| dt\_promised                         | DATE              | Promise date (UTC).                            |
| dt\_delivered                        | DATE              | Delivery date (UTC).                           |
| late\_risk\_score                    | NUMERIC(6,4)      | Predictive late risk (0..1).                   |
| risk\_bucket                         | TEXT              | low/med/high/very\_high.                       |
| orchestrator\_action                 | TEXT              | no\_change/proactive\_comms/expedite\_upgrade. |
| orchestrator\_reason                 | TEXT              | Why action chosen.                             |
| orchestrated\_promise\_ts            | TIMESTAMPTZ (UTC) | Adjusted promise (if available).               |

## fact\_carrier\_event\_enriched

| Column                                  | Type     | Description        |
| --------------------------------------- | -------- | ------------------ |
| (all columns from fact\_carrier\_event) |          |                    |
| marketplace                             | TEXT     | Market.            |
| prime\_flag                             | SMALLINT | 1/0 at order time. |
| dt\_event                               | DATE     | Event date (UTC).  |
| dt\_order                               | DATE     | Order date (UTC).  |

## fact\_wismo\_contact\_enriched

| Column                                  | Type     | Description         |
| --------------------------------------- | -------- | ------------------- |
| (all columns from fact\_wismo\_contact) |          |                     |
| marketplace                             | TEXT     | Market.             |
| prime\_flag                             | SMALLINT | 1/0 at order time.  |
| dt\_contact                             | DATE     | Contact date (UTC). |
| dt\_order                               | DATE     | Order date (UTC).   |

## fact\_nps\_post\_delivery\_enriched

| Column                                       | Type     | Description        |
| -------------------------------------------- | -------- | ------------------ |
| (all columns from fact\_nps\_post\_delivery) |          |                    |
| marketplace                                  | TEXT     | Market.            |
| prime\_flag                                  | SMALLINT | 1/0 at order time. |
| dt\_nps                                      | DATE     | NPS date (UTC).    |
| dt\_order                                    | DATE     | Order date (UTC).  |

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

## Keys & Relationships (quick map)

* `dim_customer( customer_id )` ← `fact_session.customer_id`, `fact_order.customer_id`, `fact_fulfillment_enriched.customer_id`
* `dim_product( asin )` ← `fact_browse.asin`, `fact_pdp.asin`, `fact_review_interaction.asin`, `fact_order.asin`
* `fact_session( session_id )` ← `fact_search.session_id`, `fact_browse.session_id`, `fact_pdp.session_id`, `fact_help.session_id`, `fact_ces_pre_purchase.session_id`, `session_scored.session_id`
* `fact_checkout_session( checkout_session_id )` ← `fact_checkout_step.checkout_session_id`, `fact_payment.checkout_session_id`, `fact_order.checkout_session_id`
* `fact_order( order_id )` ← `fact_payment.order_id`, `fact_ces_purchase.order_id`, `fact_fulfillment.order_id`, `fact_carrier_event.order_id`, `fact_wismo_contact.order_id`, `fact_nps_post_delivery.order_id`

> **Timezone note:** All `*_ts` are stored as **UTC**. “`*_local_ts`” columns are localized view columns—still TIMESTAMPTZ, but include the market’s offset (derived from `marketplace → tz` map).
