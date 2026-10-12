# Data Modeling Assignment: Bean There, Modeled That

**Course:** Data Modeling & Analytics Engineering
**Assignment Type:** Individual Homework
**Estimated Time:** 8–12 hours
**Total Points:** 100

---

## 1. Scenario

**Bean There** is a growing specialty coffee chain with **12 locations** across three cities. The company started with a single café five years ago and has expanded quickly. Its data currently lives in a handful of disconnected systems:

| Source System | What It Holds |
|---|---|
| **Point-of-Sale (POS)** | Orders, line items, payments, discounts, tips |
| **Loyalty App** | Customer sign-ups, points earned/redeemed, tier status |
| **Inventory System** | Ingredient/supply stock levels, deliveries, waste logs |
| **HR / Scheduling Tool** | Employees, roles, shifts, hourly rates |
| **Product Catalog (spreadsheet)** | Menu items, categories, sizes, prices, recipes |
| **Store Master List (spreadsheet)** | Store locations, open dates, square footage, region |

Leadership has asked the analytics team to build a **central data platform** so analysts and managers can answer business questions quickly and consistently, without each person writing their own messy joins across raw systems.

**You are the data engineer and modeler.** Because no real data exists for this exercise, you will first **generate realistic raw ("bronze") data** that mimics what each source system would deliver, and then **design and build a dimensional data model** on top of it.

---

## 2. Architecture: The Medallion Pattern

This assignment follows the medallion (multi-hop) architecture:

```
 SOURCE SYSTEMS          BRONZE                 SILVER (optional)         GOLD
 ┌──────────────┐     ┌──────────────┐       ┌──────────────────┐    ┌──────────────────┐
 │ POS          │ ──▶ │ Raw, as-is   │ ──▶   │ Cleaned, typed,  │ ─▶ │ Dimensional model│
 │ Loyalty      │     │ files/tables │       │ de-duplicated    │    │ (facts + dims)   │
 │ Inventory    │     │ + ingestion  │       │ conformed        │    │ answers business │
 │ HR           │     │   metadata   │       │                  │    │ questions        │
 │ Spreadsheets │     └──────────────┘       └──────────────────┘    └──────────────────┘
 └──────────────┘
   ▲ You GENERATE this    ▲ Part B           ▲ Bonus / folded into Part F    ▲ Parts C–F
```

| Layer | Characteristics | Your Responsibility |
|---|---|---|
| **Bronze** | Raw, immutable, exactly as delivered by the source. Messy. Includes ingestion metadata. No cleaning or business logic. | **Generate it** (Part B) |
| **Silver** *(optional)* | Cleaned, de-duplicated, typed, standardized | Handle inside your transformations (Part F) or as a bonus |
| **Gold** | Business-ready dimensional model | **Design and build it** (Parts C–F) |

---

## 3. Business Questions to Answer

Your gold-layer data model **must** support all of the following questions. Think carefully about what data each one requires: the grain, the measures, and the dimensions to slice by.

### Sales & Revenue
1. What are total sales (revenue) by day, week, month, and year?
2. Which **products** and **product categories** generate the most revenue and the highest profit margin?
3. How do sales compare **across stores, cities, and regions**?
4. What are the **peak hours and days of the week** for each store?
5. How effective are **discounts and promotions**? Do discounted orders have higher or lower average ticket sizes?

### Customers & Loyalty
6. What is the **average order value** for loyalty members vs. non-members?
7. How many **new vs. returning customers** do we get each month?
8. Which loyalty **tier** produces the most revenue per customer?
9. What is the **customer retention rate** (customers who purchase again within 30 days)?

### Operations & Labor
10. What is the **sales per labor hour** for each store, by shift?
11. Which employees (or roles) are on shift during the highest-volume periods?

### Inventory & Waste
12. What is the **waste percentage** by ingredient and by store?
13. Which stores are most frequently **running low** on key ingredients?
14. What is the **theoretical cost of goods sold (COGS)** per drink, based on its recipe?

> ⚠️ **Important:** Your generated bronze data must be rich enough to answer **all 14 questions**. If your data can't support a question (e.g., no weekend/weekday variation for Q4, no tier changes for Q8), your generator needs more work.

---

## 4. Bronze Data Specification (Source Schemas)

These are the **target schemas your generator must produce**. Treat them as the *shape* of each source system's export. You may **add columns** you find useful (document them), but you may not remove any. Remember that bronze data is raw: values may arrive as strings, formats may be inconsistent, and there will be defects (see Part B, Requirement 5).

### 4.1 POS System *(delivered as CSV)*

**`pos_orders`**
| Column | Type | Notes |
|---|---|---|
| order_id | INT | Unique order identifier |
| store_id | INT | Store where order was placed |
| employee_id | INT | Cashier who rang the order |
| customer_id | INT, nullable | Null for guest checkout |
| order_timestamp | DATETIME | When order was placed |
| order_channel | VARCHAR | `in_store`, `mobile_app`, `drive_thru` |
| subtotal | DECIMAL | Before discounts and tax |
| discount_total | DECIMAL | |
| tax | DECIMAL | |
| tip | DECIMAL | |
| payment_method | VARCHAR | `cash`, `credit`, `debit`, `app_wallet` |

**`pos_order_items`**
| Column | Type | Notes |
|---|---|---|
| order_item_id | INT | |
| order_id | INT | FK to `pos_orders` |
| product_id | INT | |
| size | VARCHAR | `small`, `medium`, `large`, or null |
| quantity | INT | |
| unit_price | DECIMAL | Price at time of sale |
| item_discount | DECIMAL | |
| promo_code | VARCHAR, nullable | |
| modifiers | VARCHAR, nullable | e.g., "oat milk, extra shot" |

### 4.2 Loyalty App *(delivered as JSON Lines)*

**`loyalty_members`**
| Column | Type | Notes |
|---|---|---|
| customer_id | INT | |
| first_name, last_name | VARCHAR | |
| email | VARCHAR | |
| birth_date | DATE | |
| signup_date | DATE | |
| signup_store_id | INT | |
| current_tier | VARCHAR | `bronze`, `silver`, `gold` |

**`loyalty_tier_history`**
| Column | Type | Notes |
|---|---|---|
| customer_id | INT | |
| tier | VARCHAR | |
| effective_date | DATE | |
| end_date | DATE, nullable | Null = current tier |

**`loyalty_transactions`**
| Column | Type | Notes |
|---|---|---|
| loyalty_txn_id | INT | |
| customer_id | INT | |
| order_id | INT, nullable | Links to POS order |
| txn_type | VARCHAR | `earn`, `redeem`, `expire`, `bonus` |
| points | INT | |
| txn_date | DATE | |

### 4.3 Product Catalog & Recipes *(delivered as CSV, "spreadsheet exports")*

**`products`**
| Column | Type | Notes |
|---|---|---|
| product_id | INT | |
| product_name | VARCHAR | |
| category | VARCHAR | `espresso`, `brewed`, `tea`, `pastry`, `sandwich`, `merch` |
| subcategory | VARCHAR | |
| is_seasonal | BOOLEAN | |
| launch_date | DATE | |
| discontinued_date | DATE, nullable | |
| base_price | DECIMAL | List price (changes over time) |

**`recipes`**
| Column | Type | Notes |
|---|---|---|
| product_id | INT | |
| size | VARCHAR | |
| ingredient_id | INT | |
| quantity_used | DECIMAL | |
| unit_of_measure | VARCHAR | |

### 4.4 Inventory *(delivered as CSV)*

**`ingredients`**
| Column | Type | Notes |
|---|---|---|
| ingredient_id | INT | |
| ingredient_name | VARCHAR | |
| ingredient_type | VARCHAR | `dairy`, `coffee_bean`, `syrup`, `bakery`, `packaging` |
| unit_of_measure | VARCHAR | |
| supplier_id | INT | |
| unit_cost | DECIMAL | Changes over time |

**`inventory_snapshots`** *(taken nightly per store)*
| Column | Type | Notes |
|---|---|---|
| snapshot_date | DATE | |
| store_id | INT | |
| ingredient_id | INT | |
| quantity_on_hand | DECIMAL | |
| reorder_threshold | DECIMAL | |

**`inventory_waste_log`**
| Column | Type | Notes |
|---|---|---|
| waste_id | INT | |
| waste_date | DATE | |
| store_id | INT | |
| ingredient_id | INT | |
| quantity_wasted | DECIMAL | |
| waste_reason | VARCHAR | `expired`, `spilled`, `remade_drink`, `damaged` |

### 4.5 Stores, Employees & Scheduling *(delivered as CSV)*

**`stores`**
| Column | Type | Notes |
|---|---|---|
| store_id | INT | |
| store_name | VARCHAR | |
| address, city, state, zip | VARCHAR | |
| region | VARCHAR | |
| open_date | DATE | |
| square_footage | INT | |
| has_drive_thru | BOOLEAN | |

**`employees`**
| Column | Type | Notes |
|---|---|---|
| employee_id | INT | |
| first_name, last_name | VARCHAR | |
| hire_date | DATE | |
| termination_date | DATE, nullable | |
| role | VARCHAR | `barista`, `shift_lead`, `store_manager` |
| home_store_id | INT | |
| hourly_rate | DECIMAL | Changes over time |

**`shifts`**
| Column | Type | Notes |
|---|---|---|
| shift_id | INT | |
| employee_id | INT | |
| store_id | INT | |
| shift_start | DATETIME | |
| shift_end | DATETIME | |
| hourly_rate_at_shift | DECIMAL | |

---

## 5. Your Tasks

### Part A: Requirements Analysis (10 points)

1. For **each of the 14 business questions**, identify:
   - The **business process** it relates to (e.g., Retail Sales, Inventory, Labor)
   - The **measures/metrics** needed
   - The **dimensions** needed to slice or filter the data
   - The **grain** of data required to answer it
2. Present this as a **bus matrix** (business processes as rows, conformed dimensions as columns).

> 💡 **Tip:** Do this *before* you write your generator. Knowing what questions must be answerable tells you what patterns your synthetic data needs to contain.

---

### Part B: Generate the Bronze Data (20 points)

Write a **reproducible data generator** that produces raw bronze files for all source tables in Section 4.

#### Requirement 1: Tooling & Reproducibility
- Use **Python** (recommended: `pandas`, `numpy`, `faker`), **SQL**, or another language approved by your instructor.
- The generator must run **end-to-end from a single command** (e.g., `python generate_bronze.py`).
- Use a **fixed random seed** (configurable via a command-line argument or config constant) so that the same seed always produces identical output.
- Include a `README` with setup instructions, dependencies, and run commands.

#### Requirement 2: Minimum Volume & Time Span

| Table | Minimum Size |
|---|---|
| `stores` | 12 stores across **3 cities** and at least 2 regions |
| `employees` | 60 (with some terminated/hired during the period) |
| `products` | 40 across all 6 categories |
| `ingredients` | 25 across all 5 ingredient types |
| `recipes` | Every drink/food product has a recipe for each size sold |
| `loyalty_members` | 2,000 |
| `pos_orders` | **50,000+** over **at least 12 consecutive months** |
| `pos_order_items` | 120,000+ (avg. 2+ items per order) |
| `shifts` | Realistic coverage for every store and operating day |
| `inventory_snapshots` | One row per store × ingredient × day |
| `inventory_waste_log` | 5,000+ |
| `loyalty_transactions` | Consistent with loyalty orders |

#### Requirement 3: Realistic Business Patterns
Random uniform noise is **not** acceptable. Your data must embed meaningful patterns so that the business questions produce interesting answers. At minimum, build in:

- **Time-of-day patterns:** morning rush (7–9 AM), smaller afternoon bump, quiet evenings
- **Day-of-week patterns:** weekday vs. weekend differences, and drive-thru stores behaving differently from walk-in-only stores
- **Seasonality:** iced drinks sell more in summer; hot drinks and seasonal items (e.g., pumpkin spice) peak in fall/winter
- **Store variation:** stores differ in volume, product mix, and customer base; **at least one store opens mid-period**
- **Loyalty behavior:** roughly 35–50% of orders are linked to a loyalty member; loyalty members order more often and have different basket sizes; some members change tiers during the period
- **Price changes:** at least **3 products** change `base_price` mid-period (and `unit_price` on order items must reflect the price in effect at the time of sale)
- **Cost changes:** at least **3 ingredients** change `unit_cost` during the period
- **Wage changes:** at least **5 employees** receive raises (reflected in `hourly_rate_at_shift`)
- **Product lifecycle:** at least **2 seasonal products** launched and discontinued within the period
- **Promotions:** at least **4 promo codes** with defined active periods
- **Labor/sales relationship:** staffing levels should loosely follow demand, but with some stores over- or under-staffed
- **Inventory behavior:** quantities decline with sales (according to recipes), replenish on delivery days, and occasionally fall below the reorder threshold

#### Requirement 4: Referential Integrity
Unless a defect is *intentionally injected* (Requirement 5), all foreign keys must be valid, order totals must reconcile with their line items (`subtotal` = sum of `quantity × unit_price`), and timestamps must fall within store operating periods.

#### Requirement 5: Bronze Characteristics & Intentional Data Defects
Real bronze data is messy. Your generator must **deliberately inject at least 8 different types of defects**, each at a configurable rate (e.g., 0.5%–3% of rows). Choose from the list below or invent your own:

| # | Defect Type | Example |
|---|---|---|
| 1 | **Duplicate records** | Same `order_id` delivered twice (retry/replay) |
| 2 | **Inconsistent casing / whitespace** | `credit`, `Credit`, `CREDIT `, ` credit` |
| 3 | **Mixed date/time formats** | `2025-03-14 08:15:00`, `03/14/2025 8:15 AM`, `14-Mar-25` |
| 4 | **Numbers stored as strings** | `"$4.50"`, `"4,50"` |
| 5 | **Orphaned foreign keys** | `product_id` that doesn't exist in `products` |
| 6 | **Unexpected nulls** | missing `waste_reason`, missing `size` on a drink |
| 7 | **Negative or zero quantities** | refunds/voids stored as `quantity = -1` |
| 8 | **Late-arriving records** | loyalty transaction dated before its order exists |
| 9 | **Logically impossible values** | `shift_end` earlier than `shift_start`; `tip` greater than `subtotal` |
| 10 | **Overlapping history** | tier history rows with overlapping date ranges |
| 11 | **Schema drift** | a column renamed or added partway through the period |
| 12 | **Outliers** | a 500-unit quantity or $9,999 price |

> 💡 **These defects are the point.** Your gold model and transformation logic in later parts must handle them. Expect to find and fix problems you created yourself.

#### Requirement 6: Bronze File Layout & Ingestion Metadata
- Organize files by **source system and table**, partitioned by ingestion date or month. For example:
  ```
  bronze/
  ├── pos/
  │   ├── pos_orders/ingest_month=2025-01/pos_orders_20250131.csv
  │   └── pos_order_items/ingest_month=2025-01/...
  ├── loyalty/
  │   └── loyalty_transactions/ingest_month=2025-01/loyalty_transactions_20250131.jsonl
  ├── inventory/
  ├── hr/
  └── reference/        (products, recipes, stores, ingredients)
  ```
- Use **at least three different file formats** across the sources (e.g., CSV, JSON Lines, Parquet).
- Add these **ingestion metadata columns** to every bronze table:

| Column | Description |
|---|---|
| `_source_system` | e.g., `pos`, `loyalty_app`, `inventory`, `hr` |
| `_source_file` | Name of the file the record came from |
| `_batch_id` | Identifier for the load batch |
| `_ingested_at` | Timestamp the record landed in bronze |

- **Do not clean, de-duplicate, or standardize anything** in bronze. Bronze is a faithful copy of what the source sent.

#### Requirement 7: Load Bronze into a Database *(recommended)*
Load your bronze files into **raw tables** (e.g., `bronze.pos_orders`) in a database such as DuckDB, SQLite, PostgreSQL, or Snowflake/BigQuery (if available). Provide the load script. This is what your Part F transformations will read from.

#### Part B Deliverables
1. **Generator source code** (and README)
2. **Bronze data dictionary:** each table, each column, its format *as delivered*, and which defects can appear there
3. **Row-count summary** for every bronze table
4. **Data defect log:** for each injected defect, state the table, column, defect type, injection rate, and the **actual count** injected
5. **Sample output:** the first 10 rows of each table (screenshot or pasted)
6. **Profiling queries/script** demonstrating that your volume and referential-integrity requirements are met (aside from intentional defects)
7. The generated data itself *only if the total size is under 50 MB* (otherwise the generator + seed is sufficient)

---

### Part C: Conceptual & Logical Model (20 points)

1. Create an **Entity-Relationship Diagram (ERD)** or **star/snowflake schema diagram** showing all of your fact and dimension tables and the relationships between them.
2. For **each fact table**, explicitly state:
   - The **grain** (one row represents ___)
   - The type of fact table (transaction, periodic snapshot, accumulating snapshot, or factless)
   - Which measures are **additive, semi-additive, or non-additive**
3. For **each dimension table**, list its attributes and identify the **primary key** (surrogate key) and **natural/business key**.

---

### Part D: Physical Model & Source-to-Target Mapping (15 points)

1. Write **`CREATE TABLE`** statements (standard ANSI SQL or your preferred dialect) for **all** gold tables.
2. Include:
   - Primary keys and foreign keys
   - Appropriate data types
   - `NOT NULL` constraints where appropriate
   - Audit columns you believe are appropriate (e.g., `created_at`, `updated_at`)
3. Include a **date dimension** and a **time-of-day dimension** (or justify why you combined or omitted either).
4. Provide a **source-to-target mapping (STTM)** table for **every gold table**:

| Gold Table | Gold Column | Bronze Source Table | Bronze Column | Transformation Rule |
|---|---|---|---|---|
| `dim_store` | `store_name` | `stores` | `store_name` | `TRIM`, title-case |
| `fact_sales` | `net_line_amount` | `pos_order_items` | `quantity`, `unit_price`, `item_discount` | `quantity * unit_price - item_discount` |

5. In the STTM, include a **"Defects handled"** note that references the defect IDs from your Part B defect log (e.g., "Handles Defect #1: duplicates removed via `ROW_NUMBER()` on `order_id`").

---

### Part E: Design Decisions (15 points)

Answer each of the following in a short paragraph (3–6 sentences). Justify your reasoning with respect to the business questions.

1. **Grain:** Why did you choose the grain you did for your main sales fact table? What would be lost if you chose a coarser grain (e.g., one row per order)?
2. **Slowly Changing Dimensions:** Which dimensions need history tracking, and which SCD type (1, 2, 3, etc.) did you choose for each? Consider at minimum: *loyalty tier*, *product price*, *employee hourly rate*, and *ingredient cost*.
3. **Guest customers:** Orders can have a `NULL` customer_id. How did you handle this in your model?
4. **Modifiers:** Item modifiers are stored as free text (e.g., "oat milk, extra shot"). How would you model them? Consider a bridge table, junk dimension, or separate fact table.
5. **Recipes & COGS:** How does your model make it possible to calculate theoretical COGS per drink and compare it to actual waste?
6. **Labor vs. Sales:** Sales and labor are recorded at different grains and times. How does your model allow "sales per labor hour" to be calculated?
7. **Bad data:** Pick three defects from your Part B defect log. For each, explain how your pipeline handles it (reject, quarantine, fix, or flag), and why that choice is appropriate for the business.
8. **Trade-offs:** Identify one design decision where you considered two reasonable options. What were they, and why did you pick one?

---

### Part F: Build the Gold Layer & Validate (15 points)

1. **Transform and load** bronze data into your gold model using SQL (or SQL + Python). At a minimum, build:
   - `dim_date`
   - `dim_store`
   - `dim_product` (with SCD handling for price changes)
   - **Your main sales fact table**
   - **One additional fact table** of your choice (labor, inventory snapshot, waste, etc.)
2. Your transformation logic must **handle the defects** you identified in Part B (de-duplication, standardization, type casting, orphan handling, etc.). Include any silver-layer cleanup steps as CTEs, staging views, or separate scripts.
3. Run **data quality reconciliation checks** and report results, including:
   - Gold row counts vs. bronze row counts, with the difference explained by defects you removed or quarantined
   - Total revenue in the gold fact table vs. total revenue derived from bronze (and an explanation of any difference)
   - Zero orphaned foreign keys in the gold fact tables (or an explicit "unknown member" strategy)
4. Write SQL queries **against your gold model** that answer **five** business questions of your choice from Section 3, including at least:
   - One from **Sales & Revenue**
   - One from **Customers & Loyalty**
   - One from **Operations & Labor** or **Inventory & Waste**

   For each query, state which business question it answers, provide the SQL, and **show a sample of the actual output** (top 5–10 rows).

> 💡 **Tip:** If a question is hard to answer with your model, that's a sign your design may need revisiting. Note any modifications you made as a result.

---

### Part G: Reflection (5 points)

In 1–2 paragraphs, reflect on:
- What was the most challenging part of this project: generating realistic data, modeling it, or cleaning it?
- What did building the bronze data teach you about why source systems are hard to model?
- If Bean There doubled in size and added **online ordering and delivery partners** (e.g., third-party delivery apps), how would you extend both your ingestion layer and your model?

---

## 6. Deliverables

Submit a **single PDF or Markdown report** plus a project folder (or repo link) containing code:

| # | Deliverable | Part | Format |
|---|---|---|---|
| 1 | Requirements analysis & bus matrix | A | Table(s) |
| 2 | Bronze generator code + README | B | `.py` / `.sql` etc. |
| 3 | Bronze data dictionary, row counts, defect log, sample rows, profiling results | B | Tables / screenshots |
| 4 | ERD / star schema diagram | C | PNG/PDF or Mermaid/dbdiagram.io code |
| 5 | Fact and dimension specifications | C | Tables or lists |
| 6 | DDL script | D | `.sql` |
| 7 | Source-to-target mapping | D | Table |
| 8 | Design-decision answers | E | Written paragraphs |
| 9 | Transformation/load scripts | F | `.sql` / `.py` |
| 10 | Reconciliation checks & validation queries with output | F | Code + results |
| 11 | Reflection | G | Written paragraphs |

**Suggested project structure:**
```
lastname_firstname_coffee_data_model/
├── README.md
├── report.md (or report.pdf)
├── generator/
│   ├── generate_bronze.py
│   ├── config.yaml
│   └── requirements.txt
├── bronze/                 (only if < 50 MB)
├── sql/
│   ├── 01_load_bronze.sql
│   ├── 02_create_gold_ddl.sql
│   ├── 03_transform_gold.sql
│   ├── 04_reconciliation_checks.sql
│   └── 05_validation_queries.sql
└── diagrams/
```

### Suggested Tools
- **Data generation:** Python with `pandas`, `numpy`, `faker`
- **Local database:** [DuckDB](https://duckdb.org) (recommended: fast, zero-setup, reads CSV/JSON/Parquet directly), SQLite, or PostgreSQL
- **Diagrams:** [dbdiagram.io](https://dbdiagram.io), [draw.io](https://app.diagrams.net), [Lucidchart](https://www.lucidchart.com), [Mermaid](https://mermaid.js.org)

---

## 7. Constraints & Assumptions

- You **may** use a **dimensional (Kimball-style) approach**, a **Data Vault** approach, or a **normalized (3NF)** approach for the gold layer, but you must **justify your choice**. A star schema is strongly recommended.
- Your bronze layer **must** remain raw: no cleaning logic is allowed in the generator's *output* beyond what a real source system would plausibly emit.
- Any assumption you make about missing information (e.g., how tax is allocated, how promotions are structured, how store hours work) must be **explicitly documented** in an "Assumptions" section.
- Naming conventions must be **consistent** (e.g., `bronze_` / `dim_` / `fact_` prefixes or schemas, `snake_case`).
- Use **surrogate keys** for all gold dimensions.

---

## 8. Grading Rubric

| Category | Points | Excellent (full credit) | Needs Improvement |
|---|---|---|---|
| **A. Requirements & Bus Matrix** | 10 | All 14 questions analyzed; bus matrix is accurate and reveals conformed dimensions | Missing questions; dimensions/grain unclear |
| **B. Bronze Data Generation** | 20 | *See sub-rubric below* | |
| **C. Conceptual/Logical Model** | 20 | Clear diagram; grain stated for every fact table; correct additivity; complete dimension attributes | Grain unstated or inconsistent; missing relationships |
| **D. Physical Model & STTM** | 15 | Valid, runnable DDL; correct keys & types; complete mapping that references defects | Syntax errors; missing keys; incomplete mapping |
| **E. Design Decisions** | 15 | Thoughtful, justified answers tied to business needs and observed data defects | Vague or unjustified answers |
| **F. Gold Build & Validation** | 15 | Transformations run, handle defects, reconcile to bronze; queries correct with real output | Doesn't run; defects ignored; no reconciliation |
| **G. Reflection** | 5 | Insightful; demonstrates understanding of the full pipeline and extensibility | Superficial |
| **Total** | **100** | | |

### Part B Sub-Rubric (20 points)

| Criterion | Points | What We're Looking For |
|---|---|---|
| **Completeness & volume** | 5 | All tables generated; minimum row counts and time span met; schemas match Section 4; ingestion metadata present |
| **Realism of patterns** | 5 | Time-of-day, seasonality, store variation, loyalty behavior, price/cost/wage changes all visibly present in the data (show evidence, such as a chart or query) |
| **Defect injection & documentation** | 5 | At least 8 defect types; configurable rates; defect log accurately matches actual counts |
| **Reproducibility & documentation** | 5 | Single-command run; fixed seed gives identical output; clear README; data dictionary complete; file layout and formats as specified |

### Common Deductions
- Generator is not reproducible (no seed, or output differs between runs): **−3 pts**
- Bronze data is "too clean" (fewer than 8 defect types): **−1 pt per missing type, up to −5**
- Business question cannot be answered from your generated data: **−2 pts each**
- Order totals don't reconcile to line items (outside intentional defects): **−2 pts**
- No grain stated for a fact table: **−3 pts each**
- Business question not answerable by the gold model: **−2 pts each**
- Natural keys used as primary keys in dimensions: **−2 pts**
- Inconsistent naming conventions: **−2 pts**
- Missing assumptions section: **−3 pts**

---

## 9. Hints & Things to Watch For

### Generating Data
- **Generate in dependency order:** stores → employees → products/ingredients/recipes → loyalty members → orders → order items → loyalty transactions → shifts → inventory. Many tables depend on others.
- **Derive, don't randomize, wherever possible.** Order totals come from line items; inventory depletion comes from recipes × items sold; loyalty points come from order totals. Derived data stays consistent automatically.
- **Use weighted probability distributions** (e.g., `numpy.random.choice` with `p=[...]`) for hour of day, product popularity, and channel mix.
- **Inject defects last.** Generate clean data first, then apply defect functions as a separate step. This also makes your defect log easy to produce, since you know exactly what you changed.
- **Log what you break.** Have each defect function return a count so your defect log is accurate by construction.
- **Start small.** Develop with 1 store and 1 week of data, then scale up.

### Modeling
- **Think about grain first.** Most modeling errors start with an unclear or mixed grain. Decide what one row of each fact table means *before* adding columns.
- **Not everything is a fact.** Descriptive attributes (store region, product category) belong in dimensions.
- **Be careful with order-level values.** `tip`, `tax`, and `discount_total` live on the *order*, but your fact table may be at the *line-item* level. How will you avoid double-counting when you sum them?
- **Role-playing dimensions.** Does a date or employee appear more than once in a fact table in different roles?
- **Semi-additive measures.** Inventory `quantity_on_hand` can be summed across stores but **not** across days.
- **History matters.** If a product's price changes, what price should a historical sale reflect? What about a customer who moves from Silver to Gold?
- **Unknown members.** What happens when a fact row references a dimension member that doesn't exist (orphan) or is null (guest)?
- **Conformed dimensions** like `dim_date`, `dim_store`, and `dim_product` should be reusable across multiple fact tables.

---

## 10. Bonus (up to +10 points)

Pick **one or more**:

1. **(+4) Incremental bronze delivery:** Make your generator produce **daily incremental batches** (including late-arriving and corrected records), and write an incremental (merge/upsert) load into gold.
2. **(+4) Explicit silver layer:** Build and document a distinct silver layer (typed, de-duplicated, standardized tables) between bronze and gold, with a quarantine table for rejected records.
3. **(+3) Promotion analysis:** Add a `dim_promotion` dimension and fact-level support for measuring **promotion lift** (sales during promo vs. baseline).
4. **(+3) Aggregate table:** Design an **aggregate table** or **materialized view** to speed up daily sales by store and category, and explain when you'd use it.
5. **(+3) Automated data quality suite:** Implement at least five automated data quality tests (e.g., uniqueness, not-null, referential integrity, accepted values, row-count reconciliation) that fail loudly, using SQL, dbt tests, or Great Expectations.

---

## 11. Academic Integrity

This is an **individual assignment**. You may discuss general concepts with classmates, but your generator code, diagrams, DDL, written answers, and queries must be your own work. Two students' generators should **not** produce identical datasets for the same seed. If you use AI tools, online references, or textbooks (e.g., *The Data Warehouse Toolkit* by Kimball & Ross, or *Fundamentals of Data Engineering* by Reis & Housley), **cite them** in a short "References" section.

---

## 12. Submission

- **Due:** *[Insert date and time]*
- **Submit via:** *[Insert LMS / email / GitHub repo link]*
- **File naming:** `lastname_firstname_coffee_data_model.zip` (or repo link)
- **Late policy:** *[Insert policy, e.g., −10% per day late]*

---

*Good luck, and may your joins be ever clean and your grain ever consistent!* ☕
