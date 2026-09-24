# EDA with SQL Server: sales, customers, products

This repo holds SQL scripts that explore sales data and produce two repeatable reports for customers and products. The database is SQL Server, database name `DataWarehouseAnalytics`, schema `gold`.

## What you get

14 scripts in `scripts/`, numbered in run order:

| File | Question it answers |
| ---- | ------------------- |
| `00_init_database.sql` | Creates the database, tables, and loads the CSVs. |
| `01_database_exploration.sql` | What tables and columns exist. |
| `02_dimensions_exploration.sql` | Which countries, categories, subcategories, and product names exist. |
| `03_date_range_exploration.sql` | First and last order date, span in months, oldest and youngest customer age. |
| `04_measures_exploration.sql` | Headline totals: sales, quantity, average price, orders, products, customers. |
| `05_magnitude_analysis.sql` | Totals grouped by country, gender, category, customer. |
| `06_ranking_analysis.sql` | Top 5 and bottom 5 products, top 10 and bottom 3 customers. |
| `07_change_over_time_analysis.sql` | Monthly sales, customer count, and quantity. |
| `08_cumulative_analysis.sql` | Yearly running total of sales and moving average price. |
| `09_performance_analysis.sql` | Per product, sales vs. its own average and vs. prior year. |
| `10_data_segmentation.sql` | Product cost bands and customer segments (VIP, Regular, New). |
| `11_part_to_whole_analysis.sql` | Share of total sales by category, in percent. |
| `12_report_customers.sql` | Customer view `gold.report_customers`, one row per customer. |
| `13_report_products.sql` | Product view `gold.report_products`, one row per product. |

## Data model

Star schema in schema `gold`. Grain of `fact_sales` is one row per order line.

`gold.dim_customers(customer_key, customer_id, customer_number, first_name, last_name, country, marital_status, gender, birthdate, create_date)`

`gold.dim_products(product_key, product_id, product_number, product_name, category_id, category, subcategory, maintenance, cost, product_line, start_date)`

`gold.fact_sales(order_number, product_key, customer_key, order_date, shipping_date, due_date, sales_amount, quantity, price)`

Joins:

```sql
fact_sales.customer_key = dim_customers.customer_key
fact_sales.product_key  = dim_products.product_key
```

Source CSVs for the three tables are in `datasets/csv-files/`:

- `gold.dim_customers.csv`
- `gold.dim_products.csv`
- `gold.fact_sales.csv`

The folder also holds bronze and silver CSVs from earlier stages and two expected report outputs (`gold.report_customers.csv`, `gold.report_products.csv`). Scripts 00 to 13 read only the three `gold.*` files listed above.

## Requirements

- SQL Server 2019 or later (uses `DATETRUNC`, `FORMAT`, `TOP`, `DATEDIFF`, `GETDATE()`).
- SQL Server Management Studio or Azure Data Studio.
- `BULK INSERT` permission on the server.

## Setup

1. Clone the repo and open SQL Server Management Studio.
2. Open `scripts/00_init_database.sql`.
3. Change the three file paths in the `BULK INSERT` statements to the location of the CSVs on your machine. The checked-in paths point to `C:\sql\sql-data-analytics-project\datasets\csv-files\...` and will fail on other machines. The load skips the header row with `FIRSTROW = 2`.
4. Run `00_init_database.sql`. Record row counts for the three tables to confirm the load.

Warning: script 00 drops database `DataWarehouseAnalytics` if it exists and recreates it. All data in that database is deleted. Back up before you run it.

## How to run

Run the scripts in numeric order. Scripts 01 to 04 only read metadata and totals, so they are safe to rerun. Scripts 05 to 11 join fact and dimension tables. Scripts 12 and 13 drop and recreate views, so they depend on script 00 having completed.

```text
00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07 -> 08 -> 09 -> 10 -> 11 -> 12 -> 13
```

Time queries in 07, 08, 09, 12, and 13 skip rows where `order_date IS NULL`. Those rows are counted separately as a quality check. Order counts use `COUNT(DISTINCT order_number)`. Customer counts use `COUNT(customer_key)` on `dim_customers` for all customers and `COUNT(DISTINCT customer_key)` on `fact_sales` for ordering customers.

## Reports

### Customer report

`gold.report_customers` (script 12) returns one row per `customer_key` with:

- Identifiers: `customer_key`, `customer_number`, `customer_name`, `age`, `age_group` (Under 20, 20-29, 30-39, 40-49, 50 and above).
- Behavior: `last_order_date`, `recency` in months since last order, `lifespan` in months between first and last order.
- Totals: `total_orders`, `total_sales`, `total_quantity`, `total_products`.
- Ratios: `avg_order_value = total_sales / total_orders`, `avg_monthly_spend = total_sales / lifespan`. When `total_orders = 0`, the value is 0. When `lifespan = 0`, monthly spend equals `total_sales`.

Segments:

- VIP: `lifespan >= 12` and `total_sales > 5000`
- Regular: `lifespan >= 12` and `total_sales <= 5000`
- New: `lifespan < 12`

### Product report

`gold.report_products` (script 13) returns one row per `product_key` with:

- Identifiers: `product_key`, `product_name`, `category`, `subcategory`, `cost`, `last_sale_date`.
- Behavior: `recency_in_months` since last sale, `lifespan` in months between first and last sale.
- Totals: `total_orders`, `total_sales`, `total_quantity`, `total_customers`, `avg_selling_price = ROUND(AVG(sales_amount / NULLIF(quantity, 0)), 1)`.
- Ratios: `avg_order_revenue = total_sales / total_orders`, `avg_monthly_revenue = total_sales / lifespan`. When `total_orders = 0`, the value is 0. When `lifespan = 0`, monthly revenue equals `total_sales`.

Segments, evaluated in this order:

- High-Performer: `total_sales > 50000`
- Mid-Range: `total_sales >= 10000` (so 10000 to 50000 once High-Performer rows are taken out)
- Low-Performer: `total_sales < 10000`

## Project structure

```text
EDA-SQL/
  scripts/
    00_init_database.sql
    01_database_exploration.sql
    ...
    12_report_customers.sql
    13_report_products.sql
  datasets/
    csv-files/
      gold.dim_customers.csv
      gold.dim_products.csv
      gold.fact_sales.csv
  docs/
    Project Roadmap.pdf
    business_requirements.pdf
```

Full business context, KPI formulas, and acceptance checks are in `docs/business_requirements.pdf`.

## Known issue

`scripts/12_report_customers.sql` is missing a comma after `total_products` in the final `SELECT` (lines 96-97). The view fails to create until you add it:

```sql
total_products,
lifespan,
```

## Troubleshooting

- `BULK INSERT` reports path not found: the path in script 00 must be a path the SQL Server service account can read, not only your user account. Copy the CSVs to a server-local folder and update the three paths.
- `BULK INSERT` reports access denied: run Management Studio as administrator or grant read rights on the folder to the service account.
- `DATETRUNC` not recognized: you are on SQL Server 2016 or older, or on another engine. Use 2019 or later.
- Views 12 and 13 return no rows: script 00 did not load data, or `order_date` is null for all rows. Check row counts in the three base tables first.

## Limits

- Sample data only. Do not use the totals for production decisions.
- SQL Server syntax only. The scripts will not run unchanged on Postgres, MySQL, or SQLite.
- `recency` and `age` use `GETDATE()`, so they shift each day you run them.
