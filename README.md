# Quantium Data Analytics

**English** | [简体中文](README.zh-CN.md)

This project analyzes chip sales and customer purchasing behavior to identify the most valuable customer segments, evaluates whether a store-layout trial improved sales and customer counts against comparable stores, and presents the findings to support category management decisions.

---

### Code and Resources Used

**Environment:** Python 3.13.9; pandas, numpy, matplotlib, scipy, openpyxl and mlxtend 0.25.0.

---

### Task 1 - Data preparation and customer analytics

Analyze the client's transactions and customer attributes to identify chip-purchasing behaviors and propose testable commercial actions.

#### Data Cleaning:

- Read `QVI_purchase_behaviour.csv` (72,637 customer rows) and `QVI_transaction_data.xlsx` (264,836 transaction rows). Before joining on `LYLTY_CARD_NBR`, check that customer keys are unique and non-null, all transactions have a matching customer, and the join preserves transaction row count and total sales.
- Convert the Excel date serials using 1899-12-30 as the origin. The observation period is 2018-07-01 to 2019-06-30; 2018-12-25 has no transactions.
- Exclude 18,094 Salsa rows because this task analyzes chips. Inspect the two 200-pack transactions from one customer and remove those two identified outliers, while retaining legitimate purchases of five packs. This leaves 246,740 product rows.
- Standardize variant brand names in `Cleaned_Brand_Names` and extract pack sizes from product names.
- Identify a purchase using `STORE_NBR + LYLTY_CARD_NBR + DATE + TXN_ID`. Counting product rows as purchases slightly overstates transaction frequency.

#### Data Analysis on Customer Segments:

- Group customers by `LIFESTAGE` and `PREMIUM_CUSTOMER`. Calculate total sales, unique customers, unique purchases, packs bought, transactions per customer, packs and sales per transaction, sales per customer, weighted price per pack, and the share of customers with at least two purchases.
- Decompose sales as **customers × transactions per customer × sales per transaction**. This separates a large customer base from frequent purchasing or higher spending per purchase.
- Compare brand and pack-size preferences by segment. An exploratory row-level Welch test compares spending among young and midage singles/couples; repeated purchases by the same customer mean its p-value should not be read as a customer-level causal result.

| Sales rank | Customer segment | Total sales | Customers | Purchases/customer | Packs/purchase | Repeat-customer share |
| ---: | --- | ---: | ---: | ---: | ---: | ---: |
| 1 | Older Families / Budget | $156,863.75 | 4,611 | 4.624 | 1.963 | 79.0% |
| 2 | Young Singles/Couples / Mainstream | $147,582.20 | 7,917 | 2.461 | 1.859 | 63.1% |
| 3 | Retirees / Mainstream | $145,168.95 | 6,358 | 3.126 | 1.895 | 72.3% |

![Sales by lifestage](graphs/module1_sales_by_lifestage.png)

Additional charts: [customers by lifestage](graphs/module1_customers_by_lifestage.png) and [packs per customer by segment](graphs/module1_units_per_customer.png).

#### Insights:

- Older Families / Budget are the largest sales contributor. Their 4.624 unique purchases per customer help explain sales despite a smaller customer base than the next two segments.
- Young Singles/Couples / Mainstream have the largest customer count among the three leaders; Retirees / Mainstream also benefit from a large customer base. Older and Young Families lead on packs bought per customer.
- Kettle is the most frequently purchased brand across segments. Doritos is second for **Young Singles/Couples / Mainstream**. The most common pack sizes are 175g, followed by 150g.
- These are observed purchasing patterns. Promotions targeting purchase frequency, basket size or brand preference should be evaluated separately for incremental sales and margin.

---

### Task 2 - Experimentation and uplift testing

Assess whether stores 77, 86 and 88 performed differently during the 2019-02 to 2019-04 layout trial, using comparable stores as benchmarks.

#### Monthly metrics and eligible control stores:

- Use the provided `QVI_data.csv` as supplied. It has 264,834 rows and already excludes the two 200-pack outliers, but still includes Salsa. Thus Task 2's store totals use a broader product scope than Task 1's chip-only totals.
- Aggregate by store and month: total sales, distinct customers, unique transactions per customer, packs per transaction, and average sales per pack. Unique transactions use the same composite purchase key described in Task 1.
- Require a complete 12-month observation history for candidate stores. Choose controls from the seven pre-trial months, 2018-07 through 2019-01; exclude all three trial stores from the candidate pool.

#### Control-store selection:

- Compare each trial and candidate store's **full seven-month time series** for sales and customer count, matching observations by month before calculating correlations.
- For each metric, combine the seven-month Pearson correlation (mapped from [-1, 1] to [0, 1]) with a month-aligned scale-similarity score: `exp(-mean(abs(trial - control)) / mean(trial))`. Weight these equally, then average the sales and customer scores. A higher score means a closer benchmark by this defined rule.

| Trial store | Selected control | Pre-trial sales correlation | Pre-trial customer correlation |
| ---: | ---: | ---: | ---: |
| 77 | 233 | 0.904 | 0.990 |
| 86 | 155 | 0.878 | 0.943 |
| 88 | 237 | 0.308 | 0.947 |

Store 237 has the highest combined similarity score for store 88, but its sales correlation is only 0.308. Another candidate, store 40, has negative pre-trial correlations on both sales (-0.148) and customer count (-0.129). Store 88 therefore needs a control-choice sensitivity check.

#### Sales and customer trial results:

- Scale each selected control's metric by the trial/control total ratio over the seven pre-trial months. Join the scaled benchmark and the actual trial store by **store pair and month**, not row position.
- For each month, compute `trial / scaled control - 1`. The effect estimate is the change in the **mean monthly gap** between pre-trial and trial periods. The table also shows the three-month aggregate uplift, a descriptive quantity distinct from the gap-change estimate.
- Quantify uncertainty with a 20,000-draw monthly bootstrap interval and a two-sided exact permutation over the 120 possible placements of three trial months among ten observed months. Adjust the six store-by-metric p-values using Holm's method. With only seven pre-trial and three trial months, these tests are exploratory and do not establish causality.

| Trial | Control | Metric | Trial-period aggregate uplift | Mean monthly gap change | Exact p | Holm-adjusted p |
| ---: | ---: | --- | ---: | ---: | ---: | ---: |
| 77 | 233 | Sales | +26.2% | +29.6 percentage points | 0.0500 | 0.2000 |
| 77 | 233 | Customers | +23.1% | +26.6 percentage points | 0.0500 | 0.2000 |
| 86 | 155 | Sales | +13.2% | +13.4 percentage points | 0.0167 | 0.0833 |
| 86 | 155 | Customers | +13.5% | +13.6 percentage points | 0.0083 | 0.0500 |
| 88 | 237 | Sales | +12.1% | +12.7 percentage points | 0.0583 | 0.2000 |
| 88 | 237 | Customers | +6.4% | +6.5 percentage points | 0.0583 | 0.2000 |

Trial-versus-scaled-control charts:

| Trial store | Sales | Customers |
| ---: | --- | --- |
| 77 | [Sales chart](graphs/module2_sales_trial_77.png) | [Customer chart](graphs/module2_customers_trial_77.png) |
| 86 | [Sales chart](graphs/module2_sales_trial_86.png) | [Customer chart](graphs/module2_customers_trial_86.png) |
| 88 | [Sales chart](graphs/module2_sales_trial_88.png) | [Customer chart](graphs/module2_customers_trial_88.png) |

#### Interpretation:

- Stores 77 and 86 have positive descriptive sales and customer gaps versus their selected controls. After six-test adjustment, only store 86's customer result is approximately at the 0.05 threshold. The available evidence does not establish a statistically reliable increase in both metrics for either store.
- Store 88's estimate depends strongly on its benchmark: sales uplift is +12.1% versus store 237, +4.3% versus candidate store 40, and -5.0% versus another high-ranked candidate, store 178. Store 88's own average monthly sales rose about 6.6% while store 237's fell about 4.9% from pre-trial to trial. The available data cannot determine whether layout execution, inventory, promotions or local demand caused these differences.
- Treat these findings as a reason to investigate and run a stronger follow-up trial, especially for store 88. More pre-trial history, execution records and multiple suitable controls would improve attribution.

---

### Task 3 - Analytics and commercial application

Use the Task 1 customer insights and Task 2 trial findings to brief the Category Manager through the [Task 3 presentation](Task%203%20-%20presentation%20guide_BRAND.pptx). It summarizes the leading customer segments, matched-control sales and customer results, uncertainty in the trial estimates, and practical follow-up actions. Recommendations are framed as testable hypotheses rather than proven causal effects.
