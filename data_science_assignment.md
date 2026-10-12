# Customer 360 Solutions Lab: Pricing, Retention & Lead Intelligence

**Course:** Applied Data Science / Analytics Capstone
**Assignment Type:** Team Project (3–4 students; individual variant available)
**Estimated Time:** 25–35 hours across 3–4 weeks
**Total Points:** 100
**Prerequisite:** *Bean There, Modeled That* (data modeling assignment), or the instructor-provided starter dataset

---

## 1. Client Brief

**Bean There**, the 12-store specialty coffee chain from the data modeling assignment, now has a working data platform: a bronze layer of raw source data and a gold dimensional model covering sales, loyalty, labor, and inventory.

Leadership has a new mandate: **build a Customer 360** and use it to drive three commercial priorities.

| # | Priority | Leadership's Question |
|---|---|---|
| 1 | **Pricing** | *"Where can we raise, hold, or lower prices to grow margin without losing the customers who matter?"* |
| 2 | **Retention** | *"Which loyalty members are about to stop visiting, and is it worth trying to win them back?"* |
| 3 | **Leads** | *"Of the people who show interest but haven't become loyal customers, whom should marketing and store teams focus on?"* |

> *"We don't want three disconnected spreadsheets. We want one place where a store manager, a marketer, and a pricing analyst can all see the same customer and make consistent decisions."* — Chief Commercial Officer, Bean There

**Your team's deliverable is three separate, independently validated machine learning models, integrated into one solutions dashboard/app** built on a shared Customer 360 foundation.

---

## 2. Learning Objectives

By completing this project you will be able to:

1. Frame ambiguous business problems as well-defined data science problems with measurable success criteria
2. Build a **point-in-time-correct Customer 360 feature table** that avoids data leakage
3. Develop, validate, and compare models for **price elasticity and optimization**, **churn/retention prediction**, and **lead scoring**
4. Translate model outputs into **decisions, actions, and expected financial impact**
5. Integrate multiple models into a single **application** with a coherent user experience and a shared decision layer
6. Apply MLOps and responsible-AI practices: model cards, monitoring, guardrails, and fairness checks

---

## 3. Solution Architecture

```
┌────────────────────┐
│  GOLD LAYER        │   (from the data modeling assignment)
│  fact_sales, dims, │
│  loyalty, labor... │
└─────────┬──────────┘
          │  + new sources: marketing_leads, marketing_touches, price_experiments
          ▼
┌────────────────────────────────────────────────────────────┐
│  CUSTOMER 360 FEATURE LAYER                                │
│  customer_360_snapshots  (one row per customer per month)  │
│  product_store_week      (one row per product×store×week)  │
└───────┬──────────────────────┬─────────────────────┬───────┘
        ▼                      ▼                     ▼
┌───────────────┐     ┌────────────────┐    ┌────────────────┐
│ MODEL 1       │     │ MODEL 2        │    │ MODEL 3        │
│ PRICING       │     │ RETENTION      │    │ LEADS          │
│ elasticity +  │     │ churn risk +   │    │ conversion     │
│ optimization  │     │ value at risk  │    │ propensity     │
└───────┬───────┘     └───────┬────────┘    └───────┬────────┘
        ▼                     ▼                     ▼
  scores_pricing        scores_retention       scores_leads        ← model output tables
        └──────────────────────┼─────────────────────┘
                               ▼
                ┌──────────────────────────────┐
                │ NEXT-BEST-ACTION DECISION    │   ← reconciles the three models
                │ LAYER                        │
                └──────────────┬───────────────┘
                               ▼
                ┌──────────────────────────────┐
                │ ONE SOLUTIONS APP            │
                │ Executive • Customer 360 •   │
                │ Pricing • Retention • Leads  │
                │ • Model Health               │
                └──────────────────────────────┘
```

**Architectural rules:**
- The three models are **separate**: each has its own training pipeline, evaluation, artifacts, and model card.
- Models **share** the Customer 360 feature layer and are **combined** only in the decision layer and the app.
- The app **reads precomputed scores**. It does not train models at load time.

---

## 4. Data Foundation

### 4.1 Starting Data
Use the gold layer from your data modeling assignment (or the instructor-provided starter package: *[insert link]*). You will need at minimum: sales at line-item grain, customers/loyalty members, loyalty tier history, products, stores, dates, promotions, and recipe-based unit costs.

### 4.2 Required Data Extensions
Your original data has no marketing or pricing-experiment information. **Extend your bronze generator** (or use the instructor's extension package) to add the three sources below. Follow the same bronze principles: raw, messy, with ingestion metadata, and a defect log.

**`marketing_leads`** *(CRM / marketing platform export)*
| Column | Type | Notes |
|---|---|---|
| lead_id | INT | |
| created_at | DATETIME | |
| lead_source | VARCHAR | `app_download`, `receipt_email_capture`, `website_form`, `referral`, `local_event` |
| email_hash | VARCHAR | Hashed, never raw email |
| nearest_store_id | INT | |
| campaign_id | VARCHAR, nullable | |
| device_type | VARCHAR | `ios`, `android`, `web` |
| first_offer_type | VARCHAR, nullable | e.g., `free_drink`, `bogo`, `none` |
| customer_id | INT, nullable | Populated *if and when* the lead converts to a loyalty member |
| converted_at | DATETIME, nullable | Date of conversion |

**`marketing_touches`** *(email / push / SMS platform)*
| Column | Type | Notes |
|---|---|---|
| touch_id | INT | |
| lead_id | INT, nullable | |
| customer_id | INT, nullable | |
| channel | VARCHAR | `email`, `push`, `sms` |
| campaign_id | VARCHAR | |
| offer_type | VARCHAR, nullable | `none`, `percent_off`, `free_item`, `bonus_points` |
| sent_at | DATETIME | |
| opened_at, clicked_at | DATETIME, nullable | |
| is_control_group | BOOLEAN | `TRUE` = held out, did not receive the offer |
| redeemed_order_id | INT, nullable | Order that redeemed the offer |

**`price_experiments`** *(pricing team's test log)*
| Column | Type | Notes |
|---|---|---|
| experiment_id | INT | |
| product_id | INT | |
| size | VARCHAR | |
| store_id | INT | |
| week_start | DATE | |
| price_multiplier | DECIMAL | e.g., `0.95`, `1.00`, `1.05`, `1.10` |

### 4.3 Requirements for the Extended Generator
To make modeling meaningful, your data must contain **learnable signal**:

- **Price variation:** at least **8 products** have price changes or randomized store×week price tests spanning at least ±10% around list price
- **Heterogeneous elasticity:** elasticity differs by product category (for example, pastries more elastic than espresso drinks). Store the **true parameters in a separate `ground_truth.yaml`** that you do *not* use during modeling. Use it afterward to check whether your models recover reality.
- **Churn dynamics:** members have individual purchase cadences; some gradually slow down before lapsing; discount sensitivity and tenure influence lapse risk
- **Retention offers:** at least 2 retention campaigns with **randomized control groups** (to enable ROI measurement and uplift analysis)
- **Lead behavior:** conversion depends on source, first offer, engagement (opens/clicks), time-to-first-touch, and proximity to a store, with noise
- **Defects:** inject at least **4 new defect types** into the new sources (e.g., duplicate leads with different email casing, touches with `opened_at` before `sent_at`, experiments with multiplier = 0, leads converted to nonexistent customers)

---

## 5. The Customer 360 Foundation

Build two shared feature tables that all three models draw on.

### 5.1 `customer_360_snapshots`
**Grain:** one row per customer (loyalty members *and* identified leads) per **monthly snapshot date**.

| Feature Group | Example Features |
|---|---|
| **Profile** | tenure_days, signup_source, signup_store, home_store (most frequent), age_band, current_tier, days_in_tier |
| **RFM** | recency_days, orders_30d / 90d / 365d, spend_30d / 90d / 365d, avg_inter_purchase_days, std_inter_purchase_days |
| **Basket behavior** | avg_order_value, items_per_order, share_of_orders_with_food, category_mix (% spend by category) |
| **Time & channel** | channel_mix (in-store / mobile / drive-thru), daypart_mix (morning/afternoon/evening), weekend_share, distinct_stores_visited |
| **Loyalty** | points_balance, points_earned_90d, points_redeemed_90d, redemption_rate, tier_changes_12m |
| **Price & promo** | discount_order_share, avg_discount_pct, promo_redemption_rate, spend_share_on_discounted_items |
| **Engagement** | emails_sent_90d, open_rate_90d, click_rate_90d, last_engagement_days |
| **Trend features** | spend_90d vs. prior_90d ratio, visit-frequency slope, change in avg_inter_purchase_days |
| **Model outputs** *(appended later)* | churn_prob_30d, value_at_risk, price_sensitivity_score, lead_score, next_best_action |

### 5.2 `product_store_week`
**Grain:** one row per product × size × store × week. Contains units sold, average realized price, list price, promo flags, unit cost, store attributes, seasonality fields, and lagged demand. Used by the pricing model.

### 5.3 Point-in-Time Correctness (Critical)
Every feature for a snapshot dated *T* must be computed using **only data strictly before *T***. Labels are computed from data **after** *T*.

> ⚠️ **Data leakage is the most common reason models look great in a notebook and fail in production.** Part B requires you to prove your features are leak-free.

### 5.4 Data Splits
- **Never** use random train/test splits for customer or demand data.
- Use **time-based splits**: for example, train on the earliest snapshots, validate on the middle, test on the final 2 months.
- Customers appear in multiple snapshots; for model evaluation, make sure the same customer-time period never appears in both train and test.

---

## 6. The Three Use Cases

Each use case must be developed as an **independent model** with its own problem framing, features, validation, and model card.

---

### 6.1 Use Case 1: Pricing — Elasticity & Price Optimization

**Business problem.** Bean There last reviewed menu prices over a year ago. Raising prices too much loses volume; too little leaves margin on the table. Leadership wants evidence-based recommendations by product and size.

**Decision supported.** *For each product-size, should we raise, hold, or lower the price, and by how much?*

**Modeling task.** Estimate **price elasticity of demand** and use it to **recommend prices that maximize gross margin** under business guardrails.

| Step | Requirement |
|---|---|
| **1. Baseline** | Log-log regression (`ln(units) ~ ln(price) + controls`) with product and store fixed effects. Report elasticities with confidence intervals by category. |
| **2. Advanced model** | Choose one: hierarchical/mixed-effects model, or gradient-boosted demand model with counterfactual price simulation. Explain why it's better (or not) than the baseline. |
| **3. Causal credibility** | Use the **randomized price tests** in `price_experiments` to validate. Compare observational elasticity estimates against experiment-based estimates and explain any gaps (promotion timing, seasonality, and store selection can all bias observational estimates). |
| **4. Optimization** | For each product-size, find the price that maximizes `(price − unit_cost) × predicted_units`, subject to guardrails. |

**Required guardrails (constraints) for optimization:**
- Max price change of **±10%** per review cycle
- Prices end in an approved ending (e.g., `.25`, `.50`, `.75`, `.95`)
- **Size ladder** preserved (small < medium < large) with a minimum step between sizes
- Minimum gross margin % per category
- No store-level price differences larger than a configurable limit

**Evaluation.**
| Type | Metrics |
|---|---|
| Demand forecast accuracy | WAPE and bias on held-out weeks |
| Elasticity quality | Sign sanity (should be negative), confidence interval width, agreement with experiment results, recovery of `ground_truth.yaml` |
| Business impact | Simulated change in units, revenue, and gross margin; sensitivity analysis under ±25% elasticity error |

**Required outputs.** `scores_pricing` table: product, size, current price, recommended price, elasticity (with CI), expected Δunits / Δrevenue / Δmargin (with ranges), confidence rating, and the guardrail(s) that bound the recommendation.

**Customer-level link.** Also produce a **customer price-sensitivity score** (e.g., from discount-order share, response to past promotions) that feeds back into `customer_360_snapshots`. Pricing *recommendations* are at the product/store level, not personalized per customer (see Section 9).

**Common pitfalls.** Promotions being run on slow days (confounding); seasonality masquerading as price effects; ignoring cannibalization between products (e.g., a cheaper small latte pulling sales from a medium); treating a noisy elasticity as precise.

---

### 6.2 Use Case 2: Retention — Churn Risk & Value at Risk

**Business problem.** Acquiring a new customer costs far more than keeping one, but blanket discounts erode margin and reward customers who would have stayed anyway.

**Decision supported.** *Which at-risk loyalty members should receive a retention intervention, and what kind?*

**Defining churn (a graded decision).** You must **define and justify** churn from the data. Start with this working definition and refine it:

> *Among members who were active at snapshot date T (≥2 lifetime orders and ≥1 order in the prior 90 days), **churn = no purchase in the 60 days after T**.*

Before modeling, analyze the **distribution of inter-purchase intervals** and show that your window is appropriate. A 60-day gap means something very different for a daily commuter than for a weekend visitor. Consider a customer-specific definition (e.g., gap > 2.5× the member's typical interval) and compare.

| Step | Requirement |
|---|---|
| **1. Baseline** | Regularized logistic regression using the Customer 360 features. |
| **2. Advanced model** | Gradient boosting (LightGBM, XGBoost, or scikit-learn). Tune with time-aware cross-validation. |
| **3. Calibration** | Probabilities must be **calibrated** (reliability curve + Brier score) since they will be multiplied by dollar values. |
| **4. Explainability** | Global drivers (e.g., SHAP) plus a **per-customer "why at risk"** explanation shown in the app. |
| **5. Value at risk** | `value_at_risk = churn_probability × expected_next_90_day_spend`. Model or justify the expected-spend estimate. |
| **6. Action policy** | Choose a targeting threshold using a **cost-benefit analysis** (offer cost, redemption rate, margin on retained spend), not by default 0.5 cutoffs. |

**Evaluation.**
| Type | Metrics |
|---|---|
| Ranking quality | PR-AUC, ROC-AUC, lift and cumulative gain curves, recall@top-k% |
| Calibration | Reliability plot, Brier score, expected calibration error |
| Business | Expected net value of the targeting policy vs. (a) no action, (b) blanket offer to everyone, (c) random targeting of the same size |
| Stability | Performance across time (monthly snapshots) and across segments (tier, store, region) |

**Using the control groups.** Your data includes randomized retention campaigns with holdouts. Use them to **measure the real incremental effect** of the offer (treated vs. control retention rate) and to sanity check your expected-value assumptions. *(Bonus: build an uplift model to target customers who are persuadable, not merely at risk.)*

**Required outputs.** `scores_retention` table: customer_id, snapshot_date, churn_prob_30d (or your window), risk tier, top 3 drivers, expected_next_90d_spend, value_at_risk, and recommended intervention tier.

**Common pitfalls.** Label leakage (features computed from post-snapshot data, or recency measured at the wrong date); defining churn so loosely that the model just predicts "inactive customers stay inactive"; ignoring that retained spend, not retained heads, is the business goal.

---

### 6.3 Use Case 3: Leads — Conversion Propensity & Prioritization

**Business problem.** Marketing captures thousands of leads (app downloads, receipt emails, local events, referrals), but most never become loyal customers. Follow-up capacity (emails, store-level outreach, event staff) is limited.

**Lead definition.** A *lead* is an identified contact (hashed email or app install) who is **not yet an active loyalty member** at the time of scoring.

**Conversion definition.** A lead *converts* if they become a loyalty member **and** place **≥2 orders within 60 days of lead creation** (or a definition you justify). Plain sign-up with no purchase is not a meaningful conversion.

**Decision supported.** *Which leads should get outreach this week, through which channel, and with what offer?*

| Step | Requirement |
|---|---|
| **1. Baseline** | Logistic regression or a simple rules-based score (e.g., source + first-offer weights). |
| **2. Advanced model** | Gradient boosting classifier on lead features: source, device, first offer, distance/proximity proxy to nearest store, time-to-first-touch, engagement in the first 7 days, store characteristics, local market conversion rates. |
| **3. Capacity-aware evaluation** | Evaluate **precision and lift in the top *N* leads** where *N* is the weekly outreach capacity (set by you, e.g., 200 leads/week). |
| **4. Segmentation** | Group leads into **Hot / Warm / Cold** tiers with clear, data-supported cut-offs and a recommended playbook for each. |
| **5. Expected value** | `lead_value = conversion_probability × expected_first_year_customer_value`, where expected value comes from the behavior of similar converted customers in Customer 360. |

**Evaluation.**
| Type | Metrics |
|---|---|
| Ranking quality | PR-AUC, lift@top-decile, precision@N |
| Calibration | Reliability curve, Brier score |
| Business | Expected conversions and value from working the top N leads vs. random selection and vs. the current "contact everyone in order of sign-up" approach |
| Fairness & stability | Performance across lead sources, regions, and device types |

**Time-to-act insight.** Analyze how conversion probability decays with the time since lead creation, and report the **"golden window"** in which outreach is most effective.

**Required outputs.** `scores_leads` table: lead_id, score date, conversion_probability, tier, expected_value, recommended channel, recommended offer, and top 3 drivers.

**Common pitfalls.** Using post-conversion information (e.g., orders after conversion, or the `customer_id`/`converted_at` fields themselves) as features; evaluating on all leads when only the top N will actually be contacted; mixing leads created at very different times in a single random split.

---

## 7. The Integrated Solutions App

### 7.1 Purpose
Combine the three separate models into **one application** that a store manager, marketer, and pricing analyst can all use. This is the project's capstone deliverable.

### 7.2 Technology
Any of the following is acceptable: **Streamlit** (recommended), Dash, Shiny, Gradio, or a BI tool (Power BI/Tableau) backed by the scored tables. The app must run from a **single documented command** (e.g., `streamlit run app/Home.py`) and read from your model output tables, not retrain models on load.

### 7.3 Required Pages / Views

| Page | Primary Users | Required Content |
|---|---|---|
| **1. Executive Overview** | Leadership | Headline KPIs for all three use cases; total expected impact (margin gain from pricing, value saved from retention, value gained from leads); data freshness banner |
| **2. Customer 360 Lookup** | Store managers, marketers | Search a customer or lead. See profile, RFM, purchase history timeline, tier, churn risk with "why", price sensitivity, lead score (if applicable), and the **next best action** |
| **3. Pricing Studio** | Pricing analyst | Elasticity table with CIs; recommended vs. current price; **interactive what-if simulator** (change a price, see predicted units/revenue/margin); guardrail indicators |
| **4. Retention Command Center** | Marketing, CRM | Ranked at-risk list with value at risk; filters by store, tier, risk band; driver explanations; **offer ROI calculator**; downloadable action list (CSV) |
| **5. Lead Prioritizer** | Marketing, field teams | Ranked leads; **capacity slider** that updates expected conversions/value; tier breakdown; recommended channel/offer; golden-window chart |
| **6. Model Health & Methodology** | Data science, auditors | Metrics for each model, calibration plots, drift indicators, training dates and versions, links to model cards, known limitations |

### 7.4 The Next-Best-Action (NBA) Decision Layer
The three models are separate, but their recommendations can **conflict**. Build a transparent decision layer that reconciles them into one recommended action per customer. This is the integration that makes it a *solution* rather than three dashboards.

**You must implement and document at least five reconciliation rules.** Examples:

| Situation | Reconciled Action |
|---|---|
| High churn risk **and** high price sensitivity | Offer a non-price incentive (bonus points, free upsize) rather than a percentage discount |
| High churn risk **and** the pricing model recommends raising the price on their most-purchased item | Flag the price increase for review, or exclude them from the retention offer's cost assumptions |
| Hot lead whose predicted conversion is high **without** an offer | Use a low-cost welcome message, not a costly offer (avoid subsidizing the inevitable) |
| Customer already contacted in the last *X* days | Suppress further outreach (contact-fatigue cap) |
| Low value-at-risk | No action; don't spend on low-value outreach |

The NBA layer must output a `customer_next_best_action` table with: customer/lead ID, recommended action, rationale (which models and rules triggered), expected value, and any suppression reason. The rule set should be configurable (e.g., a config file) so a business user could change thresholds without editing code.

### 7.5 App Quality Requirements
- Loads each page in a reasonable time (< 5 seconds) using precomputed scores
- Filters and drill-downs work and are consistent across pages (the same customer shows the same numbers everywhere)
- Every metric has a **plain-language tooltip or definition**
- CSV export for action lists
- A visible disclaimer that data is **synthetic**
- No raw personal data (use hashed IDs or generated names)

---

## 8. Your Tasks and Point Allocation

### Part A: Problem Framing & Business Case (10 points)
For **each of the three use cases**, produce a one-page brief:
1. Business problem and the specific **decision** the model supports
2. Stakeholders and how they will use the output
3. A **KPI tree** linking model outputs to business outcomes
4. Success metrics (technical *and* business) and what "good enough to deploy" means
5. Baseline/current approach you must beat
6. Key assumptions for the **ROI estimate** (e.g., offer cost, redemption rate, average margin) with sensible ranges
7. Risks and ethical considerations

### Part B: Customer 360 Foundation & Data Extension (15 points)
1. Extend the bronze generator with the three new sources (Section 4), including a `ground_truth.yaml`, defect injection, and an updated defect log. **(4 pts)**
2. Build and document `customer_360_snapshots` and `product_store_week` from gold + new sources. Provide a **feature dictionary** (definition, source, window, owner model(s)). **(5 pts)**
3. **Prove point-in-time correctness:** include automated tests or queries that demonstrate no feature uses data on or after its snapshot date. **(3 pts)**
4. Provide **exploratory analysis** relevant to all three use cases: purchase-interval distributions, price vs. demand patterns, lead conversion by source, and data quality findings. **(3 pts)**

### Part C: Model 1: Pricing (15 points)
Complete Section 6.1. Graded using the **Model Sub-Rubric** (Section 10).

### Part D: Model 2: Retention (15 points)
Complete Section 6.2. Graded using the **Model Sub-Rubric** (Section 10).

### Part E: Model 3: Leads (15 points)
Complete Section 6.3. Graded using the **Model Sub-Rubric** (Section 10).

### Part F: Integrated Solutions App & NBA Layer (20 points)
Complete Section 7. Graded using the **App Sub-Rubric** (Section 10).

### Part G: MLOps, Monitoring & Responsible Use (5 points)
1. A **model card** for each model (intended use, training data, metrics, limitations, ethical considerations, retraining triggers). **(2 pts)**
2. **Reproducible pipelines:** `train_all` and `score_all` commands (e.g., Makefile or scripts) that regenerate every output table from the gold layer with fixed seeds. At least **five automated tests** (e.g., feature leakage test, score range checks, no duplicate keys, guardrail satisfaction for recommended prices). **(1.5 pts)**
3. **Monitoring plan:** drift metrics on key features (e.g., PSI), performance tracking once labels mature, and clear retraining triggers; implement at least a drift check on the Model Health page. **(1 pt)**
4. A short **fairness and guardrails analysis** (Section 9). **(0.5 pt)**

### Part H: Executive Readout & Reflection (5 points)
1. A **10-minute recorded or live presentation** to Bean There leadership: the problem, what you built, your recommendations, expected impact (with ranges), risks, and next steps. **(3 pts)**
2. A **team reflection** (1–2 pages): the hardest modeling problem, a place where the data surprised you, what you'd do with 3 more months, and how you'd roll out the pricing and retention actions safely (e.g., A/B test design). **(2 pts)**

---

## 9. Responsible Use Requirements

These are graded, not optional.

- **Privacy:** Use hashed identifiers; no real personal data. Document what customer attributes you use and why.
- **Pricing fairness:** Pricing recommendations must be **product- and store-level**, not personalized to individual customers. Do not use protected or sensitive attributes (or close proxies) to set or recommend prices. Explain how you checked this.
- **Offer fairness:** Check whether retention offers and lead prioritization systematically favor or exclude any segment (age band, region, device type). Report disparities and mitigation options.
- **Contact fatigue:** Enforce a maximum number of marketing touches per customer per period in the NBA layer.
- **Transparency:** Customer-facing reasons ("why at risk") must be understandable and must not expose sensitive inferences.
- **Human in the loop:** Price changes and large campaigns require human approval. Your app should present recommendations, not auto-execute them.

---

## 10. Grading Rubric

| Category | Points |
|---|---|
| **A. Problem Framing & Business Case** | 10 |
| **B. Customer 360 Foundation & Data Extension** | 15 |
| **C. Model 1: Pricing** | 15 |
| **D. Model 2: Retention** | 15 |
| **E. Model 3: Leads** | 15 |
| **F. Integrated App & NBA Layer** | 20 |
| **G. MLOps, Monitoring & Responsible Use** | 5 |
| **H. Executive Readout & Reflection** | 5 |
| **Total** | **100** |

### Model Sub-Rubric (applies to each of Parts C, D, E: 15 points each)

| Criterion | Points | Excellent | Needs Improvement |
|---|---|---|---|
| **Problem framing & target definition** | 3 | Clear target, justified windows/definitions, tied to a specific decision | Vague target; arbitrary definitions |
| **Features & leakage control** | 3 | Features meaningful, documented, and provably point-in-time correct | Leakage present; undocumented features |
| **Modeling & validation** | 4 | Baseline + advanced model; time-based splits; correct metrics; calibration/CI where required; honest comparison | Random splits; one model; wrong or missing metrics |
| **Business translation** | 3 | Output converts to a decision and a quantified financial impact with uncertainty; cost-benefit threshold or guardrails applied | Metrics only; no decision or dollar impact |
| **Communication & code quality** | 2 | Clear notebook/report; reproducible code; model card | Hard to follow; not reproducible |

### App Sub-Rubric (Part F: 20 points)

| Criterion | Points | What We're Looking For |
|---|---|---|
| **Functionality & completeness** | 6 | All six pages implemented with the required elements; what-if simulator and capacity slider work |
| **Integration & NBA layer** | 6 | ≥5 documented reconciliation rules; configurable; produces `customer_next_best_action` with rationale; consistent numbers across pages |
| **UX & communication** | 4 | Clear layout, tooltips/definitions, sensible defaults, useful for the stated personas |
| **Reproducibility & performance** | 4 | Runs from a single command; reads precomputed outputs; loads quickly; README with screenshots |

### Common Deductions
- Random (non-temporal) train/test split: **−3 pts per model**
- Evidence of data leakage: **−3 to −5 pts per model**
- Using default 0.5 thresholds without business justification (retention/leads): **−2 pts**
- Uncalibrated probabilities multiplied by dollar values: **−2 pts**
- Pricing recommendations that violate stated guardrails: **−3 pts**
- Models trained inside the app at load time: **−3 pts**
- Models not independently reproducible (shared state, hidden dependencies): **−3 pts**
- No uncertainty ranges on financial impact estimates: **−2 pts per model**
- Missing model card: **−1 pt per model**
- Personalized pricing by individual customer: **−5 pts**

---

## 11. Deliverables

| # | Deliverable | Part |
|---|---|---|
| 1 | Three one-page problem briefs and KPI trees | A |
| 2 | Extended bronze generator, defect log, and `ground_truth.yaml` (kept separate from modeling code) | B |
| 3 | Customer 360 build scripts, feature dictionary, leakage tests, EDA notebook | B |
| 4 | Pricing model notebook/report, code, and `scores_pricing` | C |
| 5 | Retention model notebook/report, code, and `scores_retention` | D |
| 6 | Lead scoring notebook/report, code, and `scores_leads` | E |
| 7 | App source code, NBA rules config, and `customer_next_best_action` | F |
| 8 | Three model cards, tests, pipeline commands, monitoring plan, fairness analysis | G |
| 9 | Executive presentation (slides + recording/live) and team reflection | H |
| 10 | Individual contribution statement (one paragraph per team member) | All |

**Suggested repository structure:**
```
customer360_solutions_lab/
├── README.md
├── Makefile                       # train_all, score_all, run_app, test
├── requirements.txt
├── data/
│   ├── bronze/                    # generated (or generator + seed only)
│   ├── gold/                      # from the modeling assignment
│   └── features/                  # customer_360_snapshots, product_store_week
├── generator/
│   ├── generate_bronze_extension.py
│   └── ground_truth.yaml          # do not use in modeling code
├── features/
│   ├── build_customer_360.py
│   └── build_product_store_week.py
├── models/
│   ├── pricing/    (train.py, evaluate.py, optimize.py, model_card.md)
│   ├── retention/  (train.py, evaluate.py, policy.py, model_card.md)
│   └── leads/      (train.py, evaluate.py, tiers.py, model_card.md)
├── decision_layer/
│   ├── nba_rules.yaml
│   └── next_best_action.py
├── app/
│   ├── Home.py
│   └── pages/
├── tests/
├── notebooks/
├── reports/                       # briefs, readout slides, reflection
└── outputs/
    ├── scores_pricing.parquet
    ├── scores_retention.parquet
    ├── scores_leads.parquet
    └── customer_next_best_action.parquet
```

### Suggested Tools
- **Data & features:** Python, pandas/Polars, DuckDB
- **Modeling:** scikit-learn, statsmodels, LightGBM/XGBoost, SHAP, `scipy.optimize` or PuLP (price optimization), `lifelines` (survival, bonus)
- **App:** Streamlit (recommended), Plotly or Altair for charts
- **MLOps (optional):** MLflow for experiment tracking, pytest for tests, Evidently or custom PSI for drift
- **Diagrams:** draw.io, Mermaid, or dbdiagram.io

---

## 12. Team Structure & Collaboration

- **Teams of 3–4.** Suggested roles: **Pricing Lead, Retention Lead, Leads Lead,** and **Platform/App Lead**. The Customer 360 foundation (Part B) and the NBA layer are shared responsibilities.
- Every model must have **one clearly accountable owner**, but all members should be able to explain every model in the readout.
- Use version control (Git) with meaningful commits; contribution is evaluated from commit history and the individual contribution statements.
- **Individual variant:** build all three models at a reduced scope (baseline + one advanced model for each), a simplified app (four pages: Executive, Customer 360, and any two use-case pages), and a minimum of three NBA rules.

---

## 13. Hints & Things to Watch For

### Foundation
- **Build the snapshot table once, test it hard.** Every downstream model inherits its mistakes.
- **Write the leakage test first.** For each feature, assert that the max source timestamp used is strictly less than the snapshot date.
- **Keep ground truth hidden** from the modeling code, then compare your estimates against it as a final validity check.

### Pricing
- Start with the **randomized experiments**. They're your most trustworthy signal, so compare observational estimates to them.
- Always show **ranges**, not point estimates. An elasticity of −1.2 ± 0.8 is a very different decision input than −1.2 ± 0.1.
- Optimization will happily exploit model errors. Use guardrails and consider shrinking extreme recommendations toward current prices.

### Retention
- **Think in dollars, not just probabilities.** A modest-risk, high-spend customer may be worth more attention than a high-risk, low-spend one.
- "Who is at risk?" and "Who will respond to an offer?" are **different questions**. Use the control groups to see how they differ.
- Test the model on **later months** to see if performance holds up as customer behavior shifts.

### Leads
- **Order matters:** score leads using only information available at (or shortly after) creation, plus the first few days of engagement. Don't peek at the future.
- Evaluate on the **top N**, since the sales team will never contact everyone.
- Check how fast conversion decays; speed to contact can matter more than the offer itself.

### Integration & App
- Define each metric **once** (in a shared module or table) and reuse it everywhere to keep numbers consistent across pages.
- Build the NBA layer on **paper** first. List each conflict between models, decide the business logic, then code it.
- Design for the **user's question**, not the model's output. A manager asks "who should I call today?", not "what's the AUC?"

---

## 14. Bonus (up to +10 points)

Pick any:

1. **(+4) Uplift modeling:** Using the randomized retention campaigns, build an uplift (heterogeneous treatment effect) model, and show that targeting by predicted uplift beats targeting by churn risk alone.
2. **(+3) Customer lifetime value:** Add a CLV model (e.g., BG/NBD + Gamma-Gamma, or a regression-based approach) and use it to replace the expected-spend heuristic in value-at-risk and lead value.
3. **(+3) Cross-price elasticity:** Estimate cannibalization/substitution between products (e.g., cold brew vs. iced latte) and incorporate it into price optimization.
4. **(+3) Scoring API:** Expose all three models behind a small FastAPI service with an endpoint to score a single customer/lead, and have the app call it for the Customer 360 Lookup page.
5. **(+3) Rollout design:** Provide a full A/B test design for rolling out the pricing and retention recommendations: unit of randomization, sample size/power calculation, guardrail metrics, and decision criteria.
6. **(+2) Deployed app:** Deploy the app to a public or class-accessible URL with a documented deployment process.

---

## 15. Academic Integrity

Teams must submit their own work. You may discuss concepts with other teams, but code, analysis, written briefs, and presentations must be your own. Cite any datasets, libraries, tutorials, textbooks, or AI tools you used in a "References" section. Examples of useful references: *Causal Inference for the Brave and True* (Facure), *Fundamentals of Data Engineering* (Reis & Housley), and the scikit-learn and SHAP documentation. Each team member must be able to explain their team's work in a short oral follow-up if requested.

---

## 16. Submission

- **Milestone 1 (Parts A–B):** *[Insert date]*, problem briefs and Customer 360 foundation
- **Milestone 2 (Parts C–E):** *[Insert date]*, the three models
- **Final (Parts F–H):** *[Insert date and time]*, integrated app, documentation, presentation
- **Submit via:** *[Insert LMS / GitHub repo link]*
- **File naming:** `teamname_customer360_solutions_lab.zip` (or repo link)
- **Late policy:** *[Insert policy]*

---

*One customer, three questions, one source of truth. Build something a store manager would actually open on Monday morning.* ☕
