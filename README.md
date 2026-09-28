# Sales & Margin Analytics — AtScale SML model

A governed semantic model for foodservice distribution sales and margin analytics on BigQuery. Net Sales, Gross Profit, Gross Profit % and Cases Sold are defined **once** in this model (as dataset calculated columns plus one MDX ratio). Power BI, Excel, Tableau, SQL clients and AI agents all consume that single definition, so no client needs its own formula.

## Inputs

All inputs were delivered as one demo-data bundle archive and are copied verbatim into `context/`.

| File in `context/` | What it is |
|---|---|
| `sysco_sales_margin.zip` | The original input bundle archive, copied unchanged (includes CSVs, generator, load scripts) |
| `use_case.md` | Use case: scenario, governed metric definitions, 9 brief NLQs with expected answers, engineered stories, grain table, suggested hierarchies, limitations |
| `nlq_smoke_tests.sql` | NLQs as warehouse SQL with expected values (NLQs are also embedded in `use_case.md`) |
| `ddl.sql` | BigQuery DDL for the 9 tables (bundle file `bigquery/02_ddl.sql`) |
| `erd.mmd` | Mermaid ERD of the star/snowflake schema |
| `data_profile.yaml` | Column-level data profile (row counts, distinct counts, roles, additivity) |
| `manifest.json` | Machine-readable bundle summary (target, window, golden numbers) |
| `bundle_README.md` | The bundle's own README (load instructions) |
| build.yaml | (not provided) — all parameters derived or defaulted, see below |
| Target warehouse | BigQuery (from the bundle manifest and DDL) |

## Build parameters

| Parameter | Value |
|---|---|
| `unrelated_dimension_handling` | `repeat` (applied to all 15 base metrics) |
| `warehouse` | BigQuery |
| `database` (BigQuery project) | `your-gcp-project` |
| `schema` (BigQuery dataset) | `SALES_MARGIN_DEMO` |
| `model_unique_name` | `sales_margin_analytics` |
| `catalog_unique_name` | `sales_margin_analytics_catalog` |
| `currency` | USD |
| `time_window` | fiscal (July–June, named by ending year) with a secondary calendar hierarchy; data FY2024–FY2026 (2023-07-01 to 2026-06-30) |
| `use_cases_covered` | all (Sales & Margin Analytics; Plan vs Actual) |
| `use_cases_excluded` | none from the model's inputs (the bundle already excluded supply chain, rep performance, supplier income) |
| `semi_additive_default` | `sum` — both facts are transactional/flow; no semi-additive metrics |

## Assumptions and decisions

- **warehouse = BigQuery**: from `manifest.json` (`"warehouse": "bigquery"`) and BigQuery-dialect DDL (backtick-qualified names, INT64/NUMERIC types). Identifiers are lowercase, matching the DDL (Rule 1).
- **database = `your-gcp-project`**: the inputs carry only a project placeholder (`manifest.json` target.project = null). Per Rule 17 a concrete value is emitted. **Replace it with your real GCP project ID** in `connections/Connection - Sales Margin BigQuery.yml` before loading.
- **schema = `SYSCO_SALES_MARGIN_DEMO`**: the dataset name that `load.sh` and `manifest.json` default to. This is a physical warehouse identifier, kept verbatim per Rule 17. It contains the customer's name; that name is kept out of every logical name, label and description (Rule 18). Override it if you loaded to a different dataset.
- **model_unique_name = `sales_margin_analytics`**: from the use case subject ("Sales & Margin Analytics"), genericized to the domain (Rule 18). The catalog is `sales_margin_analytics_catalog` (Rule 8c).
- **time_window = fiscal**: the use case, DDL and profile define a July–June fiscal year (`fiscal_year`, `fiscal_quarter_label`, `fiscal_period_label`). All time-intelligence calcs use the Fiscal Hierarchy. The Calendar Hierarchy uses distinct level names (Calendar Year/Quarter/Month) and shares only the leaf Day level (Rule 10).
- **Prior-year calcs use ParallelPeriod** over `[Fiscal Year]`, with `parallel_periods` on Fiscal Quarter/Fiscal Period keyed by the calculated column `prior_fiscal_year = fiscal_year - 1`. The Calendar hierarchy is set up the same way with `prior_calendar_year` (Rule 13).
- **Governed measures as dataset calculated columns**: `net_sales_amount`, `gross_profit_amount` and `cases_sold` on `fact_sales` implement the use case's definitions. They are exposed as `sum` metrics, so SQL/DAX/MDX clients get identical, aggregate-friendly results. The component metrics (Gross Sales, Discounts, Returns, COGS, Cases Shipped/Returned) are exposed for drill-through transparency.
- **Gross Profit % = Divide(Gross Profit, Net Sales)**: a ratio of sums at the query grain, never an average of line margins. This is the use case's "consistency trap" (governed 17.78% vs 18.58% average-of-lines vs 20.17% on gross, FY2026).
- **Conformed date (Rule 21)**: both facts carry a single date and join the same plain Date Dimension at Day, with no role-play. `fact_sales_plan.plan_month_key` is the first day of each fiscal period, so plan rolls up correctly to Fiscal Period/Quarter/Year and "vs Plan" ratios share one time axis.
- **Plan conformance**: `fact_sales_plan` joins Organization at **Operating Company** and Product at **Category**, its natural grain. With `unrelated_dimensions_handling: repeat`, plan values repeat below those levels (Site, Subcategory, Product, Customer, Supplier, Order Channel). Compare plan at OpCo/Region/Segment × Category × fiscal period.
- **Organization snowflake**: `dim_site.opco_id` → visible `Operating Company` level (Rule 5/9). `fact_sales` joins at Site (`site_id`) and plan joins at Operating Company. `Region` is keyed `[business_segment, region]` because "Midwest"/"Southeast"/"West" exist under both U.S. Foodservice and the chain-distribution segment. A `Region Name` secondary attribute (own-column key, Rule 8b) gives a cross-segment region slicer.
- **Product snowflake**: `dim_product.category_id` → visible `Category` level. Subcategory is keyed `[category_id, subcategory]` because names such as "Vegetables" repeat across categories. A second `Brand Hierarchy` (Brand Type > Brand Name > Product) serves the private-label mix NLQ and shares the leaf Product level.
- **Customer hierarchy keys**: Customer Segment is keyed `[customer_type, customer_segment]` and Parent Account `[customer_segment, parent_account_name]`, which keeps parents single-valued.
- **is_unique_key**: set only on single-column keys that are table primary keys (date_key, opco_id, site_id, category_id, product_id, customer_id, supplier_id). The profile shows distinct_count = row_count for these (Rule 16). Not set on the degenerate Order Channel dimension.
- **Order Channel** is a degenerate dimension on `fact_sales` (`is_degenerate: true`), listed in the model's `dimensions:` block (Rule 12).
- **Top-N NLQs (6, 7)** need no ranking MDX (TopCount/Rank are unsupported). They are served by slicing Gross Profit / Net Sales by Supplier or Parent Account, with the client sorting and limiting (SQL `ORDER BY … LIMIT 10`, BI Top-N filter).
- **Brand share NLQ (3)** is served as Cases Sold by Brand Type with the client's percent-of-total. No member-specific tuple calc is emitted, which avoids hard-coding a data value into MDX.
- **Omitted columns as attributes**: `dim_customer.home_opco_id`/`primary_site_id` (the organization comes from the fact's `site_id`), `dim_product.supplier_id` (the fact carries `supplier_id`), and `dim_supplier.primary_category_id` are defined in the datasets but not exposed as attributes. `account_open_date`/`account_close_date` are not modeled. No dataset is unreferenced.
- **Calculated display columns** on `dim_date` (`calendar_quarter_key`, `calendar_quarter_label`, `calendar_month_label`, `weekend_flag`, `prior_fiscal_year`, `prior_calendar_year`), `dim_category.protein_flag` and `dim_supplier.private_label_packer_flag` give readable labels for boolean and compound values.

## Generation summary

- **Datasets:** 9. Dimension sources: `dim_date`, `dim_operating_company`, `dim_site`, `dim_category`, `dim_product`, `dim_customer`, `dim_supplier`. Facts: `fact_sales` and `fact_sales_plan`.
- **Dimensions:** 6 — Date (time; Fiscal + Calendar hierarchies), Organization (snowflake), Product (snowflake; Category + Brand hierarchies), Customer, Supplier, Order Channel (degenerate).
- **Base metrics:** 15 — Net Sales, Gross Profit, Cases Sold (governed); Gross Sales, Discounts, Returns, Cost of Goods Sold, Cases Shipped, Cases Returned; Invoice Count, Active Customers, Invoice Lines; Net Sales Plan, Gross Profit Plan, Cases Plan.
- **Calculated metrics:** 19 — Gross Profit %, Net Sales per Case, Gross Profit per Case, Discount Rate, Average Invoice Net Sales; Prior Year (Net Sales, Gross Profit, Cases Sold, GP%), YoY Growth % (Net Sales, Cases), GP% Change vs PY; Fiscal YTD (Net Sales, Gross Profit); vs Plan $ and % (Net Sales, Gross Profit), GP% Plan.
- **Model relationships:** 8 (fact_sales → Date, Organization@Site, Product@Product, Customer, Supplier; fact_sales_plan → Date, Organization@Operating Company, Product@Category).
- **Role-play prefixes:** none (conformed single date per fact).
- **Snowflake bridges:** 2 (dim_site → Operating Company; dim_product → Category).
- **Calculated columns added:** 11 (3 governed measures on fact_sales, 6 on dim_date including the two prior-year keys, 1 on dim_category, 1 on dim_supplier).

### NLQ coverage

| # | NLQ | Answered with |
|---|---|---|
| 1 | Net Sales, GP, GP% by OpCo, FY2026 | Net Sales, Gross Profit, Gross Profit % × Organization › Operating Company × Fiscal Year |
| 2 | Protein GP% FY26 vs FY25 | Gross Profit %, Gross Profit % Prior Year, GP% Change vs PY × Product › Category (or Protein Flag) |
| 3 | Private-label case share and GP% | Cases Sold, Gross Profit % × Product › Brand Type × Fiscal Year (client % of total) |
| 4 | Local vs National GP% and $/case | Gross Profit %, Net Sales per Case × Customer Type × Business Segment |
| 5 | Region with largest GP% decline, driving OpCo | GP% Change vs PY × Region › Operating Company × Customer Type, Fiscal Quarter Q3–Q4 |
| 6 | Top 10 suppliers by GP | Gross Profit, Gross Profit % × Supplier (client sort/limit) |
| 7 | Top 10 customer accounts by Net Sales | Net Sales, Gross Profit % × Customer › Parent Account (client sort/limit) |
| 8 | Monthly cases by segment | Cases Sold × Customer Segment × Fiscal Period |
| 9 | Net Sales / GP vs plan by region, FY26 H2 | Net Sales vs Plan %, Gross Profit vs Plan % × Region × Fiscal Quarter |

Golden check (FY2026, all clients must match): Net Sales $308,801,518.87 · Gross Profit $54,914,349.40 · Gross Profit % 17.78% · Cases Sold 4,216,413.

### Caveats

- Replace `database: your-gcp-project` with the real GCP project before loading. Verify that the dataset name matches your load.
- Gross Profit % Change vs PY is formatted as percent, so a value of −3.4% reads as −3.4 percentage points.
- Plan measures are only meaningful at OpCo/Category/fiscal-period grain or above (they repeat below it).
- The model has not yet been loaded into a live AtScale instance. It passed the bundled validator only.

## Reproducing this build

`context/` holds verbatim copies of every input this build consumed, including the original bundle archive. No `build.yaml` was supplied: every parameter above was derived from those inputs or defaulted, and each choice is logged under *Assumptions and decisions*. Re-running the SML model generator with the same `context/` inputs and these build parameters (in particular `unrelated_dimension_handling: repeat`, `database: your-gcp-project`, `schema: SYSCO_SALES_MARGIN_DEMO`) should produce an equivalent model.
