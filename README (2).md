# Supply Chain Delivery and Inventory Analytics
**Python + Amazon S3 + Amazon Redshift + Amazon QuickSight**

An end-to-end analytics project: raw order and inventory files cleaned with Python, staged in Amazon S3, modeled with layered SQL in **Amazon Redshift**, and reported in a three-sheet **Amazon QuickSight** dashboard with WBR-style metrics, root-cause analysis and a quantified recommendation.

**Headline finding:** 54.8% of order lines ship later than promised, and the cause is the delivery promise, not the region. First Class promises 1 day but always takes 2, so about 95% of its orders are late. Resetting that one promise to 2 days would cut overall late delivery from **54.8% to 40.1%**.

## Dashboard preview

| Business review | Delivery performance | Inventory health |
|---|---|---|
| ![Business review](dashboard/01_business_review.png) | ![Delivery performance](dashboard/02_delivery_performance.png) | ![Inventory health](dashboard/03_inventory_health.png) |

Full export: [`dashboard/supplychain_analysis_dashboard.pdf`](dashboard/supplychain_analysis_dashboard.pdf)

## Business questions

1. How reliably do orders ship against the promised delivery time, and what drives late delivery?
2. Is the problem regional, seasonal, or tied to the shipping mode?
3. How often does demand meet zero recorded inventory (stockouts), and where is stock cost concentrated?
4. What single change would improve service the most, and by how much?

## Architecture

```
DataCo orders CSV + inventory CSV
        |  Python (pandas): clean, remove PII, standardize        python/prep_dataco.py
        v
Amazon S3 (cleaned CSV files, private bucket)
        |  load
        v
Amazon Redshift Serverless
  raw:      dataco_cleaned, raw_inventory
  staging:  stg_order_lines (view)
  core:     fact_order_lines (DISTKEY order_id, SORTKEY order_ts), dim_products (DISTSTYLE ALL)
  marts:    mart_delivery_monthly, mart_region_performance, product_pareto,
            mart_inventory_health, mart_wbr_weekly, mart_mbr_monthly
        |  SPICE import
        v
Amazon QuickSight dashboard (3 sheets)
```

The WBR and MBR views use `LAG()` window functions for week-over-week and month-over-month variance.

## Pipeline steps

**1. Python data preparation (`python/prep_dataco.py`)**
- Converts the orders file from latin-1 to UTF-8.
- Keeps the 28 columns needed for analysis and drops customer names, street addresses and credentials.
- Converts order and shipping dates to ISO format.
- Trims and normalizes text fields, and translates Spanish country names to English.
- Produces `dataco_clean.csv` (180,519 rows) and `inventory_clean.csv` (4,200 rows).

**2. Amazon S3 staging**
- The cleaned files are uploaded to a private S3 bucket with Block Public Access turned on. Only the cleaned files are stored; the original file with personal data is not uploaded.

**3. Amazon Redshift modeling**
- The cleaned files are loaded into raw tables, then transformed through staging, fact and dimension tables and six reusable marts (see the architecture above).
- Fact table is distributed on `order_id` and sorted on `order_ts`; the small product dimension uses `DISTSTYLE ALL`.

**4. Amazon QuickSight reporting**
- Datasets are imported into SPICE from the Redshift tables and views through a read-only database user.
- Three sheets: Business review, Delivery performance, Inventory health.

## Key results (Jan 2015 to Sep 2017)

| Metric | Value |
|---|---|
| Orders | 57,288 |
| Sales | $34.2M |
| Profit margin | 11.98% |
| Late delivery rate (order lines) | 54.82% |
| Late rate if First Class promise were 2 days | 40.13% |
| Stockout rate (product-months with demand and zero stock) | 23.7% |
| Total inventory carrying cost (units x cost per unit, summed monthly) | $84,748 |

### Findings

**1. Late delivery is driven by shipping mode, and the promises are unrealistic.**

| Shipping mode | Promised days | Actual days | Late rate |
|---|---|---|---|
| First Class | 1 | always 2 | about 95% |
| Second Class | 2 | 2 to 6 | about 77% |
| Same Day | 0 | 0 or 1 | about 46% |
| Standard Class | 4 | 2 to 6 | about 38% |

**2. Region and time are not the cause.** Late rates are about 55% in every market (Europe, LATAM, Pacific Asia, USCA, Africa) and stay flat month to month, so the lateness is structural, not seasonal.

**3. Recommendation: reset the First Class promise from 1 day to 2 days.** Because First Class always takes 2 days, this removes nearly all of its late orders and lowers the overall late rate from 54.82% to 40.13%. This assumes actual delivery days do not change. The next step would be a controlled test on a subset of orders. Second Class (promise 2 days, actual 2 to 6) is the next promise to review.

**4. About 24% of product-months with demand had zero recorded inventory.** Stockouts were stable around 20% to 30% a month, then jumped from **11.3% in Mar 2017 to 48.8% in Apr 2017**. That month is the first one to investigate.

**5. Carrying cost is concentrated.** Three products hold about half (about 52%) of the total: Perfect Fitness Perfect Rip Deck (about $18.5K), Nike Men's Dri-FIT Victory Golf Polo (about $14.4K) and O'Brien Men's Neoprene Life Vest (about $12K). Their stock levels should be reviewed for overstock.

**6. Sales are concentrated.** Across the full dataset, the top 5 products make about 61% of sales and the top 10 about 90% (SQL view `product_pareto`).

## Metric definitions

| Metric | Definition |
|---|---|
| Late delivery % | Order lines flagged late / all order lines |
| Stockout % | Product-months with demand and zero warehouse inventory / product-months with demand (a fill-rate proxy, not true order fill rate) |
| Carrying cost | Warehouse inventory units x cost per unit, summed over months |
| Profit margin | Sum of profit / sum of order item total |

## Data

- **Orders:** DataCo Smart Supply Chain public dataset, 180,519 order lines and 118 products, Jan 2015 to Jan 2018. Source: [add the link where you downloaded it].
- **Inventory:** a separate monthly inventory file for the same 118 products (4,200 rows). It has no warehouse location, and I could not confirm its origin, so I treat it as supplementary sample data. Inventory findings show the method and are indicative, not measured operations.
- **Analysis window:** Jan 2015 to Sep 2017. From Oct 2017, every order has a single line and units and sales drop sharply, so those months look incomplete and are excluded.
- **Privacy:** customer names, addresses and credentials were removed before the data reached S3 or Redshift. The raw files are not included in this repository.

## Data quality issues found and fixed

| Issue | How I found it | Fix |
|---|---|---|
| `cost_per_unit` loaded with no decimals (scale 0), understating carrying cost by about 41% ($50,988 vs $86,431) | Reconciled dashboard totals to the source file | Recreated the table with `DECIMAL(10,4)` and reloaded |
| QuickSight import hung after rebuilding views | Read `error_message` in `sys_query_history`: `permission denied` | Recreated views lose grants after `DROP ... CASCADE`; re-granted `SELECT` and documented the order |
| Redshift `AVG()` on integer columns returns an integer | Compared averages to a pandas check | Cast to `DECIMAL` before averaging |
| Three products named differently in orders and inventory (2014 vs 2017 model year) | Compared product lists between files | Mapped them so all 118 products join |
| Spanish country names, stray whitespace in text fields | Profiling | Translated and trimmed in the Python prep script |
| Incomplete data after Sep 2017 | Trend charts and lines-per-order check | Excluded from the analysis window |

## Limitations

- No shipping cost and no warehouse location in the data, so network placement and cost-driver analysis are out of scope.
- Profit looks randomly distributed: about 18.7% of order lines are loss-making in every department, payment type and discount level, and discount rate shows no effect on profit. Margin conclusions are therefore limited.
- Delivery days are spread very evenly within each mode, which suggests the delivery data is simulated. The method carries over to real data; the specific rates may not.
- The stockout rate is a proxy based on zero recorded units in a month, not an order-level fill rate.

## How to reproduce

1. Run `python python/prep_dataco.py <path to DataCoSupplyChainDataset.csv> <output folder>` to produce `dataco_clean.csv` and `inventory_clean.csv`.
2. Upload both cleaned files to a private S3 bucket (Block Public Access on).
3. Create the raw tables in Redshift and load the files from S3. Create `raw_inventory` with `cost_per_unit DECIMAL(10,4)` before loading.
4. Run `sql/supply_chain_analysis.sql` from top to bottom (staging, fact and dimension, marts, WBR and MBR, analysis queries).
5. Grant read access to the QuickSight user:
   ```sql
   GRANT USAGE ON SCHEMA public TO quicksight_user;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO quicksight_user;
   ```
6. Connect QuickSight to Redshift, import the marts into SPICE, and filter dates to Jan 2015 to Sep 2017.

## Repository structure

```
.
|-- README.md
|-- python/prep_dataco.py
|-- sql/supply_chain_analysis.sql
`-- dashboard/
    |-- 01_business_review.png
    |-- 02_delivery_performance.png
    |-- 03_inventory_health.png
    `-- supplychain_analysis_dashboard.pdf
```

## Skills demonstrated

| Area | Where in this project |
|---|---|
| Python (pandas) | Cleaning, PII removal and standardization script |
| Amazon S3 | Private staging bucket for cleaned files |
| SQL on a columnar warehouse (Redshift) | Distribution and sort keys, window functions (`LAG`, running totals), CTEs, views |
| Data modeling and reusable datasets | Raw, staging, fact and dimension, and mart layers |
| Dashboards (QuickSight) | Three sheets, calculated fields, SPICE datasets |
| WBR and MBR reporting | Weekly and monthly variance views and KPI cards |
| Root-cause analysis | Mode, market and time ruled in or out with data |
| Quantified recommendation | First Class promise what-if: 54.8% to 40.1% late |
| Data quality and troubleshooting | Reconciliation to source, `sys_query_history`, grant management |

## Next steps

- Run a controlled test of the First Class promise change on a subset of orders.
- Add warehouse and freight cost data to analyze placement and cost drivers.
- Automate the data refresh and add alerting on late rate and stockout rate.
