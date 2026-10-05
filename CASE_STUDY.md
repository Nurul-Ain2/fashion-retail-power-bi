# When 69% of Revenue Isn't About Your Products (And Why One Trousers SKU Beats an Entire Category)

## The Business Question

A fashion e-commerce retailer asked a deceptively simple question: *"Is our revenue driven by the products we stock, or by the marketing campaigns we run?"*

On the surface, it sounds straightforward. But answering it requires solving a real data problem — one that most beginner analysts skip over, and most production systems struggle with.

## The Data

I was given 11 weeks of transaction data (April – June 2025) from a 6-country European fashion retailer: ~900 orders, ~2,250 line items, 500 products across 5 categories, 7 marketing campaigns across two sales channels (E-commerce and App Mobile).

The immediate question: Can I reliably measure how much revenue each campaign drove?

## The Interesting Challenge

Here's where it got real: **there was no direct foreign key linking sales to campaigns.**

The only shared field was a marketing channel (Email, Social Media, App Mobile, Website Banner), and campaigns ran on those channels at specific date ranges. Multiple campaigns could run on the same channel at different times.

The true match was:
```
salesitems.channel = campaigns.channel 
AND salesitems.sale_date between campaigns.start_date and campaigns.end_date
```

This is a **range join** — not something Power BI relationships handle natively. Most people either:
- Ignore the problem and get wrong numbers
- Build a slow DAX calculated column
- Or give up and do a simpler analysis

I chose a third path: **solve it in Power Query.**

Using a merge + custom filter in Power Query, I matched each sales line item to the campaign it fell within (if any), creating a clean `campaign_id` column in my fact table. Some line items had no matching campaign — those are organic sales. Others had one. This approach:
- Pushes complexity upstream (where it belongs)
- Produces a simple, clean fact table
- Made the model lightweight and queryable
- Is honest: it shows exactly which revenue is attributable to active campaign windows vs. organic

**Why this matters for your portfolio:** This isn't a tutorial exercise. This is the kind of messy-data-thinking that separates junior analysts from people who ship real dashboards.

## What the Data Showed

Two findings hit hard:

### 1. Campaigns drive 69% of revenue.

Out of $324K total revenue, **$225K came during active campaign windows.** This isn't correlation; it's a direct, attributable match. For a marketing-driven business, this is massive validation — it proves campaigns work.

The winner: **TIVA Week**, which generated $81K in revenue — nearly 36% of all campaign revenue in an 11-day window.

### 2. Category success ≠ product success.

I expected Shoes (the #1 category at $70K) to have the highest-revenue SKU. It didn't.

**Relaxed Ribbed Trousers** — a single SKU — generated $2,379 in revenue from just 36 units sold.

That's **one product outperforming an entire category** by 34×. Margin was 47.5%, and it had the lowest inventory investment (36 units). This product is a no-brainer: low inventory, high margin, high volume. Stock it.

Meanwhile, the Pants category showed 254 weeks of inventory on the shelf against only $53.8K in revenue. That's an inventory misalignment waiting to be solved.

**The insight:** Revenue concentration matters. You don't win by having 500 average products. You win by finding the 20–30 SKUs that *actually* convert and doubling down on them.

## What I'd Do With More Data

With 12+ months of history, I'd:
- Model seasonality: do campaigns lift differently in summer vs. winter?
- Build cohort retention: which campaign channels produce repeat buyers?
- Forecast inventory: predict stock-outs for high-velocity SKUs like Relaxed Ribbed Trousers
- Measure campaign carryover: does a campaign in week 1 influence sales in week 3?

11 weeks is enough to prove the concept. A year proves the strategy.

## The Portfolio Lesson

This project taught me three things:

**1. Real data is messy.** The campaign join wasn't a mistake in the data — it was a design choice reflecting how marketing and transactions actually work. Understanding that distinction is what separates "I built a dashboard" from "I solved a business problem."

**2. Specificity beats generality.** Knowing "Shoes are popular" is useless. Knowing "Relaxed Ribbed Trousers at 47.5% margin on 36 units is your cash cow" is actionable.

**3. Honest scope matters.** I didn't try to measure churn or retention on 11 weeks of data. I didn't pretend stock snapshots were time-series. I flagged limitations clearly. That honesty is what makes a portfolio piece credible.

---

## Key Metrics at a Glance

- **Total Revenue:** $324.2K
- **Total Orders:** 905
- **Average Order Value:** $358.27
- **Gross Margin:** 43.53%
- **Campaign Revenue:** $225K (69.47% of total)
- **Revenue per Customer:** $559.03
- **Orders per Customer:** 1.56
- **Top Campaign:** TIVA Week ($81K revenue)
- **Top Product:** Relaxed Ribbed Trousers ($2,379K revenue, 36 units, 47.5% margin)
- **Top Category:** Shoes ($70.07K revenue)
- **Customer Age Leader:** 26–35 age group ($69K revenue)

---

*For technical implementation details, data model architecture, and DAX measure definitions, see [README.md](README.md).*
