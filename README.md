# Fashion Retail Sales & Campaign Performance — Power BI Report

## Overview

This report analyzes online fashion retail sales from April 4 to June 17, 2025 (11 weeks) across two sales channels (E-commerce and App Mobile) and six European countries. It answers three core business questions: Which marketing campaigns drive the highest revenue and compare to organic sales? How do product categories and individual SKUs perform in revenue, profitability, and inventory efficiency? And which customer demographics (geography, age, channel preference) generate the most value? Intended for Marketing, Merchandising, and Finance leadership.

---

## Data Sources

| Table | Rows | Grain | Business Meaning |
|-------|------|-------|-----------------|
| Customers | 1,000 | One row per customer | Customer demographics (country, age range, signup date) |
| Products | 500 | One row per product (SKU) | Product catalog with pricing, cost, category, brand, size, color, gender |
| Sales | 905 | One row per order/transaction | Order headers with total amount, channel, date, customer reference |
| Sales Items | 2,253 | One row per line item within an order | Fact table; the most granular grain; includes quantity, pricing, discounts, and channel campaign attribution |
| Campaigns | 7 | One row per marketing campaign | Campaign metadata: name, channel, date range, discount type/value |
| Stock | 1,000 | One row per product per country | Inventory levels by product and country (Germany, France only); a single snapshot, not time-series data |
| Channels | 2 | One row per sales channel | Channel descriptors (E-commerce, App Mobile) |

---

## Data Model

The report uses a **star schema** with `fact_salesitems` as the central fact table, surrounded by properly configured dimensions.

**Schema Structure:**
```
                    dim_customers
                          |
    dim_products ─── fact_salesitems ─── dim_calendar
                   /       |       \
              dim_sales  dim_channels  dim_campaigns
              
    dim_stock (snapshot, related to products only)
```

**Tables & Relationships:**
- **Fact Table:** `fact_salesitems` (2,253 rows) — the analysis grain; connects to Products, Customers (via Sales), Channels, Campaigns, and Calendar
- **Sales Header:** `sales` table (905 rows) — order-level totals and customer/channel reference; bridges salesitems to customers
- **Dimensions:** Products (500), Customers (1,000), Channels (2), Calendar (built from date min/max, marked as official date table)
- **Campaigns:** 7 rows; connected via the range-join solution (see below)
- **Stock:** Snapshot only; relates to Products and their countries (Germany, France) but not Calendar

### The Campaign Join — A Deliberate Design Decision

The most complex modeling challenge was connecting sales transactions to marketing campaigns, because there is no direct foreign key. The match requires:
- `salesitems.channel_campaigns` = `campaigns.channel` AND
- `salesitems.sale_date` within `campaigns.[start_date, end_date]` (a **range join**)

**Solution:** Built a `campaign_id` column in Power Query using the range-join logic above, then created a simple one-to-many relationship `fact_salesitems[campaign_id]` → `dim_campaigns[campaign_id]`. This approach:
- Solves complexity in the ETL layer (Power Query), not DAX — cleaner separation of concerns and better performance
- Produces a clean fact table: one campaign per line item, or NULL for organic (non-campaign) sales
- Enables accurate campaign-attribution metrics: revenue during active campaign windows vs. organic sales

**Why not Option B (DAX calculated column)?** Option A is industry best practice — push transformations upstream into ETL/Power Query when possible, keeping the semantic model lightweight and the analysis grain explicit.

---

## Key Metrics & Definitions

| Measure | Definition / DAX | Business Meaning |
|---------|------------------|------------------|
| **Total Revenue** | `SUM(salesitems[item_total])` | Total order value across all transactions |
| **Total Orders** | `DISTINCTCOUNT(sales[sale_id])` | Number of unique orders/transactions |
| **Total Cost** | `SUM(salesitems[quantity]) * AVERAGE(products[cost_price])` | Total cost of goods sold |
| **Gross Margin** | `[Total Revenue] - [Total Cost]` | Revenue minus COGS; absolute profit |
| **Gross Margin %** | `DIVIDE([Gross Margin], [Total Revenue])` | Profit as a percentage of revenue |
| **Average Order Value** | `DIVIDE([Total Revenue], [Total Orders])` | Mean transaction size |
| **Campaign Revenue** | `CALCULATE([Total Revenue], NOT ISBLANK(salesitems[campaign_id]))` | Revenue from line items that matched an active campaign window (channel + date range) |
| **Campaign Revenue %** | `DIVIDE([Campaign Revenue], [Total Revenue])` | % of total revenue attributable to active marketing campaigns |
| **Organic Revenue** | `[Total Revenue] - [Campaign Revenue]` | Revenue from non-campaign sales |
| **Discount Rate %** | `COUNT(salesitems[discount_applied]) / COUNT(salesitems[item_id])` | % of line items that received a discount |
| **Discounted Line Items %** | `COUNTA(salesitems[discount_applied]) / COUNTA(salesitems[item_id])` where discounted=1 | Portion of order volume sold at discount |
| **Items per Order** | `DIVIDE([Total Quantity Sold], [Total Orders])` | Average units per transaction |
| **Orders per Customer** | `DIVIDE([Total Orders], DISTINCTCOUNT(customers[customer_id]))` | Average repeat purchase frequency |
| **Revenue per Customer** | `DIVIDE([Total Revenue], DISTINCTCOUNT(customers[customer_id]))` | Customer lifetime value (11-week window) |
| **Revenue per Unit** | `DIVIDE([Total Revenue], [Total Quantity Sold])` | Average selling price per item |
| **Revenue WoW %** | `DIVIDE([Total Revenue] - [Revenue (Prior Period)], [Revenue (Prior Period)])` | Week-over-week growth rate (directional, limited history) |
| **Sell Through %** | `VAR UnitsSOld = SUM(salesitems[quantity]) VAR StockQty = SUM(stock[stock_quantity]) RETURN IF(StockQty = 0, BLANK(), (UnitsSOld / StockQty) * 100)` | % of available inventory sold (snapshot-based, directional) |
| **Stock Coverage (Weeks)** | Inventory weeks of supply | How many weeks of historical sales volume current stock would support |

**Quality notes:**
- All calculations use `salesitems` as the fact-table grain, except customer/order counts which use DISTINCTCOUNT to avoid double-counting
- Campaign attribution uses the range-join logic: `campaign_id` is NOT BLANK only for line items where `channel_campaigns` = `campaigns.channel` AND `sale_date` within campaign date range
- Sell Through % and Stock Coverage are snapshot-based (single point in time), not time-series — directional only
- All calculations account for line-level discounts, not header-level flags, to avoid misrepresentation

---

## Known Limitations

- **Data span:** 11 weeks only (Apr 4 – Jun 17, 2025). This is insufficient for reliable churn, retention, or cohort analysis. WoW trend comparisons are directional; MoM analysis would be misleading. Seasonal patterns are not yet visible.

- **Stock data is a snapshot, not a time series.** Stock quantity represents a single point in time (presumably end of June 2025), not inventory movement across weeks. Sell-Through % and Stock Coverage metrics are therefore directional (all products cluster 12–15% sell-through), indicating relative inventory levels rather than selling velocity. To measure true inventory turnover, weekly stock snapshots would be required.

- **Stock coverage is partial geography.** Stock levels are available for Germany and France only (2 of 6 customer countries). Inventory insights do not reflect full global stock position.

- **Campaign attribution is at the channel level, not campaign-specific discounting.** The join matches `channel_campaigns` to `campaigns.channel` (Email, Social Media, App Mobile, Website Banner) and date windows; it does not capture discount_type/discount_value linkage at the transaction level. Campaign lift metrics reflect *revenue during active windows*, not causation.

- **Data quality:** All 7 files contain zero nulls. Discount percentages were converted from text ("10.00%") to numeric in Power Query.

---

## How to Use This Report

**Report Structure:**
The report has 5 main pages, accessed via tabs at the top. Use slicers on the left to filter by Channel (E-commerce / App Mobile), Country, and Month.

**1. Executive Overview** — Start here. High-level KPIs (Total Revenue $324.2K, Average Order Value $358, Gross Margin 43.5%) and weekly revenue trend. Use this to answer "How are we performing overall?"

**2. Sales Performance** — Channel and geographic deep dive. Weekly revenue trend by channel, revenue by country, and top 10 products by revenue. Filter by Channel or Country to isolate performance drivers.

**3. Campaign Effectiveness** — Campaign ROI dashboard. Shows revenue per campaign, campaign revenue as % of total (69.47% in this dataset), discount rates, and organic vs. campaign revenue split by week. Key finding: TIVA Week drove the highest revenue ($81K). Use this to evaluate marketing ROI.

**4. Product Performance** — SKU and category analysis. Revenue by category, margin vs. volume scatter plot, stock coverage by category, and detailed product table (revenue, units, margin %, sell-through %). Reveals high-efficiency products (Relaxed Ribbed Trousers: $2,379K on 36 units) and inventory misalignment (Pants: 254 weeks stock vs. $53.8K revenue).

**5. Customer Profile** — Demographics and channel preference. Revenue by age group (26–35 leads with $69K), revenue per customer ($559), orders per customer (1.56), and channel preference by age cohort. Shows which segments are most valuable.

**Key Questions Answered:**
- *Which campaigns drive the most revenue?* → Campaign Effectiveness page; filter by campaign or channel
- *Which products are most profitable?* → Product Performance page; view margin % and revenue together
- *Which customer segments spend the most?* → Customer Profile page; filter by age range or country
- *Is there inventory risk?* → Product Performance; view Stock Coverage by Category; Pants at 254 weeks signals overstock

**Recommended next actions:**
- Use Country slicer to dig into regional performance (Germany is top market at $75K)
- Compare E-commerce vs. App Mobile channel performance
- Investigate week 21 revenue spike (Executive Overview) — correlates with campaign activity
