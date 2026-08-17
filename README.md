## Data Source

**Olist Brazilian E-Commerce Public Dataset**  
[Kaggle Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

# Checkout Incentives & LTV — E-Commerce Customer Analytics Project

A full-funnel analysis of the Brazilian **Olist e-commerce dataset**, moving from raw data auditing through Customer Lifetime Value (LTV) modeling, checkout incentive economics, product-category retention analysis, and a machine-learning system to predict repeat-purchase behavior.

The project answers one central business question:

> **Can Olist increase customer lifetime value by selectively incentivizing the right customers to make a second purchase — and if so, who, when, and how much should the incentive be?**

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Notebook 01 — Data Audit](#notebook-01--data-audit)
3. [Notebook 02 — LTV Modeling](#notebook-02--ltv-modeling)
4. [Notebook 03 — Checkout Incentive Analysis](#notebook-03--checkout-incentive-analysis)
5. [Notebook 04 — High-Value Customer Conversion & Incentive Economics](#notebook-04--high-value-customer-conversion--incentive-economics)
6. [Notebook 05 — Product Category Retention Analysis](#notebook-05--product-category-retention-analysis)
7. [Notebook 06 — Customer Repeat-Purchase Prediction (ML)](#notebook-06--customer-repeat-purchase-prediction-ml)
8. [Notebook 07 — Executive Synthesis & Retention Strategy](#notebook-07--executive-synthesis--retention-strategy)
9. [Cross-Notebook Business Recommendations](#cross-notebook-business-recommendations)
10. [Key Numbers Cheat Sheet](#key-numbers-cheat-sheet)
11. [Limitations (Project-Wide)](#limitations-project-wide)
12. [Tech Stack](#tech-stack)

---

## Project Overview

This project analyzes e-commerce customer and order data to investigate four pillars:

1. **A/B Testing** — statistical rigor, hypothesis testing, multiple-testing correction, experiment design
2. **Customer Lifetime Value (LTV) Modeling** — historical value quantification and segmentation
3. **Checkout Incentives** — economics, ROI, and targeting of discount campaigns
4. **Product Sense** — translating statistical evidence into prioritized, budget-aware business decisions rather than blanket strategies

**Dataset:** Olist Brazilian e-commerce dataset — 9 relational tables (Customers, Geolocation, Orders, Order Items, Payments, Reviews, Products, Sellers, Category Translation), 99,441 order-linked customer records, 96,478 delivered orders.

**Guiding principle throughout the project:** flag data anomalies rather than blindly delete them; use `customer_unique_id` (not `customer_id`) for all customer-level metrics; prefer targeted, evidence-based incentive strategies over universal discounting; treat every "opportunity" number as a scenario estimate, not a causal guarantee, until validated by a controlled experiment.

---

## Notebook 01 — Data Audit

**Purpose:** Establish data quality, structure, and integrity before any modeling.

### Dataset Structure
| Dataset | Purpose |
|---|---|
| Customers | Customer info & location |
| Geolocation | ZIP-level lat/long |
| Orders | Order lifecycle & timestamps |
| Order Items | Products, sellers, price, freight |
| Payments | Payment methods, installments, values |
| Reviews | Review scores & comments |
| Products | Product characteristics |
| Sellers | Seller info & location |
| Category Translation | PT → EN category mapping |

### Key Findings

**Missing values**
- `order_approved_at`: 160 missing; `order_delivered_carrier_date`: 1,783 missing; `order_delivered_customer_date`: 2,965 missing — concentrated in non-delivered order statuses (expected).
- Review titles: 87,656 missing; review messages: 58,247 missing (optional field, expected).
- Products: 610 missing category/name/description/photo-count values; 2 missing dimensional values.
- **24 delivered orders** anomalously missing lifecycle timestamps — flagged, not deleted.

**Duplicates**
- Geolocation: 261,831 exact duplicate rows out of 1,000,163 (not auto-removed — legitimate coordinate reuse is possible).
- All other tables: 0 duplicates.
- 17,781 of 19,015 ZIP prefixes show multiple latitude values → geolocation table requires validation before geospatial use.

**Customer identity**
- 99,441 `customer_id` rows but only **96,096 unique `customer_unique_id`** values.
- 2,997 customers hold multiple `customer_id` records (max 17 for one customer).
- **Conclusion: all customer-level metrics (LTV, repeat purchases) must use `customer_unique_id`.**

**Referential integrity**
- Customer → Order: complete (0 orphans).
- Payment → Order: 1 order (ID `30710`) missing a payment record despite being delivered with $134.97 + $8.49 freight in items and a 1-star "never received" review — flagged as incomplete data, not proof of non-payment.
- Order → Item: 775 orders have no item record.
- Order → Review: 768 orders have no review (expected — reviews are optional).

**Financial consistency**
- Of 98,665 orders with both item and payment data, **98,284 (99.61%) match exactly**.
- 381 orders (0.39%) show discrepancies: mean abs. difference $8.58, median $3.77, max $182.81.
- 291 orders: payment > item total; 90 orders: payment < item total.
- Discrepancies correlate with installment count (10-installment orders average $18.81 diff; 12-installment average $24.83) and payment type (90.55% of discrepancies tied to credit cards).
- 178 of the 381 discrepant orders are under $100 — nearly half the discrepant set.

**Chronological anomalies**
- 166 orders: shipment before purchase.
- 23 orders: delivery before shipment.
- 8,320 reviews created before recorded delivery (271 before shipment, 6 before purchase) — flagged as possible survey-dispatch artifacts, not automatically invalid.

**Payment anomalies**
- 9 zero-value payment records (mostly vouchers).
- 2 credit-card records with invalid zero installments.
- 3 `not_defined` payment-type records, all tied to canceled $0 orders.

**Geographic concentration**
- SP: 40,302 customers, $5.2M sales — dominant.
- RJ: 12,384 customers, $1.8M.
- MG: 11,259 customers, $1.6M.
- These 3 states dominate both customer count and order activity; remaining 24 states trail far behind.

**Repeat purchase behavior (first look)**
- 93,099 customers made exactly 1 order (96.88%).
- 2,997 customers made 2+ orders (3.12%); max 17 orders by one customer.

### Audit Conclusion
The dataset is structurally sound for downstream modeling. No records were deleted purely for being anomalous — everything is flagged for investigation, preserving data fidelity for LTV modeling, A/B testing, incentive analysis, and segmentation.

---

## Notebook 02 — LTV Modeling

**Objective:** Quantify economic customer value and identify segments most relevant for checkout incentives. Uses **historical LTV** (total amount spent to date), not a predicted future value.

**LTV formula:** `LTV = total payment value across all recorded orders`, aggregated at `customer_unique_id` level.

### Overall LTV
| Metric | Value |
|---|---|
| Total customers | 96,096 |
| Total revenue | $16,008,872.12 |
| Average LTV | $166.59 |
| Median LTV | $108.00 |
| Min / Max LTV | $0.00 / $13,664.08 |

**Inference:** Mean > median → strongly right-skewed spending. A minority of high-spending customers pull the average up. Report both mean (aggregate forecasting) and median (typical customer).

### One-Time vs. Repeat Customers
| Type | Customers | % | Avg LTV | Median LTV |
|---|---:|---:|---:|---:|
| One-time | 93,099 | 96.88% | $161.82 | $105.70 |
| Repeat | 2,997 | 3.12% | $314.99 | $225.84 |

Repeat customers have **~1.95× higher mean LTV** and **~2.14× higher median LTV**. Only 3.12% of customers repeat — this is the central behavioral gap the whole project addresses.

### Revenue Contribution
| Type | Customer % | Revenue % |
|---|---:|---:|
| One-time | 96.88% | 94.10% |
| Repeat | 3.12% | 5.90% |

One-time customers generate most *absolute* revenue simply due to volume — but per-customer, repeat customers are far more valuable ($314.99 vs $161.82).

### LTV Distribution Bands
| Band | % of Customers |
|---|---:|
| <$50 | 16.49% |
| $50–100 | 29.71% |
| $100–250 | 38.98% |
| $250–500 | 10.15% |
| $500–1000 | 3.40% |
| $1000–2000 | 1.03% |
| $2000+ | 0.23% |

~46.2% of customers have LTV below $100; only 4.67% exceed $500.

### Revenue Concentration (Pareto)
| Segment | Revenue Share |
|---|---:|
| Top 1% | 10.48% |
| Top 5% | 27.04% |
| Top 10% | 38.51% |
| Top 20% | 53.77% |

**Business implication:** A blanket incentive strategy wastes budget on low-incremental-value customers. Target by expected value and responsiveness.

### LTV by Order Count
| Orders | Customers | Avg LTV |
|---:|---:|---:|
| 1 | 93,099 | $161.82 |
| 2 | 2,745 | $294.85 |
| 3 | 203 | $473.39 |
| 4 | 30 | $779.13 |
| 5–17 | 19 | (small-sample, not generalizable) |

Clear monotonic pattern: more purchases → higher LTV. Extreme order-count groups have tiny samples and shouldn't be generalized.

### High-Value Threshold
Threshold set at **LTV = $319.57** (approx. 90th percentile):

| Segment | Customers | Avg LTV | Revenue Share |
|---|---:|---:|---:|
| High-value | 9,611 (10%) | $641.55 | 38.52% |
| Standard | 86,485 (90%) | $113.81 | 61.48% |

### Customer Lifetime & Return Timing
- One-time customers: lifetime = 0 days by definition.
- Repeat customers: avg lifetime 87.31 days, median 33.66 days.
- Time to second purchase: mean 80.35 days, median 27.92 days, max 608.98 days (right-skewed).

**Second-purchase timing distribution:**
| Window | % of Repeat Customers |
|---|---:|
| 1–7 days | 36.60% |
| ≤30 days (cumulative) | 51.1% |
| ≤60 days | 61.8% |
| ≤90 days | 68.7% |
| ≤180 days | 82.8% |
| ≤365 days | 97.1% |

**Business implication:** Early post-purchase weeks are a key intervention window, but a single fixed incentive window would miss many later returners.

### Four-Quadrant Incentive Segmentation
| Segment | Customers | % of Base | Avg LTV | Revenue Share |
|---|---:|---:|---:|---:|
| **High-value one-time** | 45,432 | 47.28% | $264.91 | **75.18%** |
| High-value repeat | 2,634 | 2.74% | $347.05 | 5.71% |
| Low-value one-time | 47,667 | 49.60% | $63.55 | 18.92% |
| Low-value repeat | 363 | 0.38% | $82.33 | 0.19% |

**The single most important finding of the notebook:** the **high-value one-time** segment is less than half the customer base but drives three-quarters of revenue, and has *not yet* repeated. This becomes the primary incentive target for the rest of the project.

### Notebook Limitations
This is **historical**, not predictive, LTV. It does not: predict future purchases, estimate response probability to incentives, calculate incremental revenue caused by incentives, establish causality, account for margin, or subtract incentive cost.

---

## Notebook 03 — Checkout Incentive Analysis

**Objective:** Translate Notebook 02 findings into a business-focused incentive strategy — who to target, when, and whether the economics work.

### Repeat-Purchase Timing Windows
| Window | Customers | Avg 2nd-Order Value |
|---|---:|---:|
| Same day | 276 | $134.36 |
| 1–7 days | 821 | $152.31 |
| 8–30 days | 433 | $162.08 |
| 31–60 days | 320 | $156.45 |
| 61–90 days | 208 | $151.36 |
| 91–180 days | 424 | $144.10 |
| 180+ days | 515 | $141.84 |

276 same-day repeats vs 2,721 next-day-or-later — most repeat behavior is genuine return behavior, not order-splitting. Second-order value stays fairly stable ($134–$162) across all windows, so timing doesn't strongly bias order value. Repeat behavior spans multiple horizons — a single short campaign window would miss a large share of opportunity (515 customers return after 180+ days).

### First-Order Value vs. Repeat Rate (Counter-Intuitive Finding)
| First-Order Band | Repeat Rate |
|---|---:|
| <$50 | 3.41% |
| $50–99 | 3.28% |
| $100–249 | 3.17% |
| $250–499 | 2.89% |
| $500–999 | 2.73% |
| $1,000+ | 2.03% |

**Repeat rate DECREASES as first-order value increases.** High spend does not predict repeat likelihood — high-value customers are attractive because of monetary value, not because they're more likely to return.

### Weak Differentiators (Tested, Rejected)
- **Geography:** repeat rates range 1.52%–5.26% by state, but small states are statistically unreliable; large states (SP: 3.31% on 39,150 customers) provide the strongest evidence.
- **Delivery speed:** one-time avg 12.57 days vs repeat avg 12.39 days — negligible (0.18-day) difference.
- **Review behavior:** one-time avg score 4.15 vs repeat 4.16; review rates 99.34% vs 99.23% — negligible difference.

**Conclusion:** none of these are strong targeting variables. Monetary value + purchase history remain the strongest rationale.

### Target Segment & Opportunity Sizing
Target: **45,432 high-value one-time customers** ($12.04M historical revenue, 75.18% of revenue share).

Applying the historical 3.12% repeat rate to this segment:
- Estimated conversions: **1,417**
- Avg second-order value: **$146.94**
- **Estimated incremental revenue ≈ $208,219.24**

### Conversion-Rate Scenarios
| Conversion Rate | Conversions | Est. Revenue |
|---:|---:|---:|
| 1% | 454 | $66,710.76 |
| 3% | 1,363 | $200,279.22 |
| 5% | 2,272 | $333,847.68 |
| 10% | 4,543 | $667,548.42 |

### Incentive Cost / ROI Sensitivity (2nd-order value = $146.94)
| Incentive | Net Rev/Conversion | ROI |
|---:|---:|---:|
| $5 | $141.94 | 2,838.80% |
| $10 | $136.94 | 1,369.40% |
| $25 | $121.94 | 487.76% |
| $50 | $96.94 | 193.88% |
| $75 | $71.94 | 95.92% |
| $100 | $46.94 | 46.94% |
| $125 | $21.94 | 17.55% |
| $146.94 | $0.00 | 0.00% (break-even) |

**Key insight:** ROI% is constant across conversion-rate scenarios for a fixed incentive amount — because both revenue and cost scale linearly with conversions. **Conversion rate determines scale of opportunity; incentive size determines per-conversion economics.** Bigger discounts are not automatically better — value erodes as incentive approaches the $146.94 ceiling.

### Business Recommendation
Target **high-value one-time customers only**. Test multiple incentive levels via **A/B experiment** (Control / Treatment A-low / Treatment B-moderate / Treatment C-high), measuring **incremental repeat-purchase rate vs. control** (e.g., 4.5% treatment − 3.1% control = 1.4pp uplift), not raw conversion count.

### Important Caveat
This is a **scenario model, not a causal estimate**. Historical 3.12% repeat rate and $146.94 second-order value describe what happened, not what an incentive will cause. A controlled experiment is required to establish causal incremental lift.

---

## Notebook 04 — High-Value Customer Conversion & Incentive Economics

**Objective:** Deepen the business case for targeting high-value one-time customers and formalize incentive economics.

### LTV vs. Repeat Propensity (Median Cutoff = $108)
| Segment | Customers | Repeat Rate | Avg LTV |
|---|---:|---:|---:|
| Low-value | 48,030 | 0.76% | $63.69 |
| High-value | 48,066 | **5.48%** | $269.41 |

**High-value customers repeat 7.21× more often than low-value customers** (5.48% / 0.76%). This is a stronger, more direct signal than Notebook 03's aggregate 1.95× LTV multiplier — it shows value *predicts* repeat propensity, not just repeat value.

### Target Segment Breakdown by First-Order Band
| Band | Customers | Revenue | Customer Share | Revenue Share |
|---|---:|---:|---:|---:|
| $100–249 | 32,516 | $5.25M | 71.57% | 43.60% |
| $250–499 | 8,825 | $2.98M | 19.42% | 24.73% |
| $500–999 | 2,969 | $2.02M | 6.54% | 16.80% |
| $1,000+ | 1,122 | $1.79M | 2.47% | 14.87% |

**Inference:** The segment isn't homogeneous. High-spend sub-tiers (6.54% + 2.47% of customers) contribute disproportionately (16.80% + 14.87% of revenue) — supports a tiered incentive strategy.

### Target Segment Summary
- Target customers: **45,432**
- Avg LTV: $264.91 / Median LTV: $180.21
- Customer share: 47.28% / Revenue share: **75.18%**
- High-value repeat rate: 5.48% vs low-value 0.76% (**7.21× multiple**)
- Avg second-order value (refined): **$148.51**

### Incentive Scenario Grid (revised 2nd-order value $148.51)
At **$25 incentive**, ROI is constant at **494.06%** regardless of conversion rate:
| Conversion | Conversions | Est. Revenue | Incentive Cost | Net Revenue |
|---:|---:|---:|---:|---:|
| 1% | 454 | $67,425.30 | $11,350 | $56,075.30 |
| 3% | 1,363 | $202,424.41 | $34,075 | $168,349.41 |
| 5% | 2,272 | $337,423.52 | $56,800 | $280,623.52 |
| 10% | 4,543 | $674,698.53 | $113,575 | $561,123.53 |

### Full Incentive-Cost Sensitivity Table
| Incentive | Net Rev/Conv | ROI |
|---:|---:|---:|
| $5 | $143.51 | 2,870.28% |
| $25 | $123.51 | 494.06% |
| $50 | $98.51 | 197.03% |
| $100 | $48.51 | 48.51% |
| $148.51 | $0.00 | 0.00% |

### Reference Experiment Design
- Target: 45,432 high-value one-time customers
- Assumed conversion: 5% → 2,272 conversions
- 2nd-order value: $148.51 → $337,423.52 revenue
- Incentive: $25/conversion → $56,800 cost
- **Net incremental revenue: $280,623.52 | Modeled ROI: 494.06%**

**$25 is chosen as reference treatment** because it's well below the $148.51 ceiling, leaves ~$123.51/conversion before other costs, and gives a meaningful, testable ROI signal for an A/B test vs. control.

### Caveat
Explicitly labeled a **hypothetical/scenario analysis**: the 5% conversion rate is an assumption, not a forecast; $148.51 and $25 are planning inputs; no causal claim is made.

---

## Notebook 05 — Product Category Retention Analysis

**Objective:** Determine whether retention/incentive efforts should be prioritized by product category rather than applied uniformly.

### Dataset
112,650 product-item observations across 72 categories; total item revenue (price + freight) = **$15,843,553.24**.

### Repeat-Item Baseline
- One-time items: 105,082 | Repeat items: 7,568 → **overall repeat-item share = 6.72%** (used as the baseline for all category comparisons).

### Revenue Concentration by Category
Top 10 categories (health_beauty, watches_gifts, bed_bath_table, sports_leisure, computers_accessories, furniture_decor, housewares, cool_stuff, auto, garden_tools) account for **62.32%** of total category revenue.

### High Repeat-Share Categories (raw, before filtering for sample size)
`diapers_and_hygiene` (25.64%, n=39), `arts_and_craftmanship` (20.83%, n=24), `home_appliances` (17.90%, n=771), `fashio_female_clothing` (16.67%, n=48) — **small-sample categories were flagged as unreliable despite high percentages.**

### Repeat vs. One-Time Item Value (Counter-Intuitive Finding)
High repeat share ≠ high spend per repeat item. Examples:
- `home_appliances`: 17.90% repeat share but repeat customers spend **37.91% less** per item than one-time customers.
- `furniture_bedroom`: repeat customers spend **54.86% less**.
- `small_appliances`: repeat customers spend **85.97% MORE** (positive exception).

### Statistical Testing
- **Mann–Whitney U test** per category (non-parametric, handles skew) for one-time vs repeat item revenue.
- **Benjamini–Hochberg FDR correction** applied across all category tests to control false discovery from multiple comparisons.
- 12 categories remained statistically significant post-correction (e.g., computers_accessories p=0.0003, health_beauty p=0.0003, home_appliances p=0.0009, bed_bath_table p=0.0027).

**Key inference:** statistical significance ≠ business priority. `health_beauty` is significant but below the repeat-share baseline, so it was not prioritized.

### Prioritization Framework
**High Priority** requires ALL THREE:
1. Repeat-item share above 6.72% baseline
2. Statistically significant (post-FDR)
3. Category revenue ≥ $50,000

### Final High-Priority Categories
| Category | Repeat Share | Lift | Revenue | Revenue Share |
|---|---:|---:|---:|---:|
| bed_bath_table | 10.02% | +3.30pp | $1,241,681.72 | 7.84% |
| computers_accessories | 7.05% | +0.33pp | $1,059,272.40 | 6.69% |
| furniture_decor | 9.85% | +3.13pp | $902,511.79 | 5.70% |
| fashion_bags_accessories | 12.95% | +6.23pp | $184,273.54 | 1.16% |
| home_appliances | 17.90% | +11.18pp | $94,990.43 | 0.60% |

**Combined:** 30,078 items, 2,888 repeat items, **9.60% repeat-item share** (vs 6.72% baseline), **$3,482,729.88 revenue (21.98% of total category revenue)**.

### Category-Level Business Insight
`bed_bath_table` is the strongest overall candidate — combines high repeat propensity, large volume, significant revenue ($1.24M), and statistically significant evidence, with only a modest (-4.89%) value drop per repeat item. `home_appliances` shows the strongest frequency signal (17.90% share) but the weakest per-item economics (-37.91%) — suggests frequency-based incentives there need careful margin evaluation, not blanket discounting.

**Core takeaway:** the category with the *highest repeat propensity* is not necessarily the category with the *highest monetary value per repeat purchase*. Retention strategy needs 3 dimensions: (1) likelihood of repeat, (2) commercial scale, (3) value of the resulting repeat purchase.

### Limitations
Observational (not causal); repeat status is historical only (may undercount future repeaters); analysis is item-level, not exactly equal to customer-level probability; uses revenue, not profit (no cost/margin data); statistical vs. business significance can diverge; small-sample categories intentionally excluded from prioritization.

---

## Notebook 06 — Customer Repeat-Purchase Prediction (ML)

**Objective:** Test whether **first-order behavior** can predict/rank customers by likelihood of repeat purchase, to support targeted retention campaigns.

### Pipeline
```
Customer/Order Data → Customer Identity Analysis → Repeat-Customer Definition →
First-Order Feature Engineering → Data Quality Filtering → Statistical Testing →
Multiple-Testing Correction → Feature Matrix → Train/Test Split → Multiple ML Models →
Model Evaluation → Threshold Optimization → Top-K Targeting → Baseline Comparison →
Business Recommendation
```

### Environment
Pandas, NumPy, Matplotlib, Scikit-learn, SciPy, Statsmodels, XGBoost 3.4.1, LightGBM 4.7.0, CatBoost 1.2.10, Statsmodels 0.14.6.

### Target Definition & Class Imbalance
- Unique customers: 96,096 → One-time 93,099 (96.88%) / Repeat 2,997 (3.12%).
- This is a **severely imbalanced binary classification problem**.

### Data Quality Filtering
- 707 customers had missing first-order features (82.32% tied to "unavailable" order status, 16.55% canceled) — excluded.
- Final modeling population restricted to `order_status = delivered`: **93,253 customers** (2,843 excluded from 96,096).
- Final target split: One-time 90,379 (96.92%) / Repeat 2,874 (3.08%).

### First-Order Behavior: One-Time vs. Repeat (Counter-Intuitive Finding)
| Metric | One-time | Repeat | Diff |
|---|---:|---:|---:|
| Avg first-order value | $160.65 | $146.83 | −8.60% |
| Avg items | 1.14 | 1.22 | +7.06% |
| Merchandise value | $137.88 | $124.05 | −10.03% |
| Freight | $22.77 | $22.78 | +0.06% |

**Repeat customers spent LESS on their first order but bought slightly MORE items** — reinforcing Notebook 03's finding that high initial spend ≠ high repeat likelihood, while item count is a positive signal.

### Statistical Testing (α = 0.05, FDR-corrected)
**Mann-Whitney U (numerical features):**
| Feature | Significant? |
|---|---|
| First-order items | Yes |
| Product count | Yes |
| Merchandise value | Yes |
| Freight value | No |
| Total value | Yes |

**Chi-square (categorical/temporal features):**
| Feature | Significant? |
|---|---|
| Customer state | No (p=0.2718) |
| First-order month | Yes (p<0.000001) |
| Day of week | No (p=0.9088) |
| First-order hour | Yes (p=0.000028) |

**6 features survived Benjamini-Hochberg FDR correction:** first-order items, product count, merchandise value, total value, first-order month, first-order hour.

### Modeling Setup
- Feature matrix: 93,253 × 9 features → 96.92% / 3.08% class split.
- Stratified train/test split: 74,602 train / 18,651 test.
- **Why accuracy is misleading:** predicting "always one-time" already yields 96.92% accuracy — so PR-AUC, F1, recall, Top-K lift are the real metrics of interest.

### Model Comparison
| Model | Accuracy | Precision | Recall | F1 | ROC-AUC | PR-AUC |
|---|---:|---:|---:|---:|---:|---:|
| CatBoost | 0.6121 | 0.0384 | 0.4817 | 0.0711 | **0.5770** | **0.0432** |
| XGBoost | 0.6248 | 0.0397 | 0.4817 | **0.0734** | 0.5643 | 0.0424 |
| LightGBM | 0.6620 | 0.0379 | 0.4087 | 0.0694 | 0.5571 | 0.0401 |
| Random Forest | 0.9321 | 0.0436 | 0.0574 | 0.0495 | 0.5631 | 0.0397 |
| Logistic Regression | 0.5496 | 0.0362 | 0.5304 | 0.0677 | 0.5475 | 0.0390 |
| AdaBoost | **0.9692** | 0.0000 | 0.0000 | 0.0000 | 0.5282 | 0.0332 |

**Critical inference:** AdaBoost's 96.92% accuracy comes from predicting every customer as "one-time" — 0% repeat recall, functionally useless. Random Forest's 93.21% accuracy hides only 5.74% repeat recall. **Accuracy is explicitly rejected as the primary selection metric.**

### Selected Models
- **Primary ranking model: CatBoost** — best ROC-AUC (0.5770), best PR-AUC (0.0432), best broader Top-K performance.
- **Alternative model: XGBoost** — best F1 (0.0734), strong recall (48.17%).
- **Rejected: AdaBoost** (zero recall). **Not preferred: Random Forest** (poor recall despite high accuracy).

### Feature Importance (Random Forest)
First-order freight value (0.1985), total value (0.1960), merchandise value (0.1838), hour (0.1324), month (0.1005) dominate. Note: importance = predictive contribution, not causality.

### Threshold Optimization
Lowering the classification threshold (0.50 → 0.05) pushes recall toward 100% but collapses precision toward ~3% — confirms the model is better suited to **ranking** than a fixed 0.5 cutoff. Threshold choice should depend on marketing budget and cost/value tradeoffs, not a default.

### Top-K Targeting & Lift
| Top-K | XGBoost Lift | CatBoost Lift |
|---:|---:|---:|
| 1% | 1.918× | 1.744× |
| 5% | 1.810× | 1.705× |
| 10% | 1.426× | **1.548×** |
| 20% | 1.313× | **1.461×** |
| 50% | 1.231× | 1.242× |

CatBoost outperforms XGBoost at broader targeting depths (10%+); XGBoost is stronger at the very top (1–5%).

### Baseline Comparison (Simple Rule vs. ML) — Top 10%
| Method | Capture Rate | Lift |
|---|---:|---:|
| First-order total value | 8.87% | 0.887× |
| First-order merchandise value | 9.22% | 0.922× |
| **First-order items** | **14.61%** | **1.461×** |
| XGBoost | 14.26% | 1.426× |
| CatBoost | 15.48% | 1.548× |

**Important insight:** the simple "first-order item count" rule nearly matches XGBoost and comes close to CatBoost — **complex ML is not dramatically superior to a simple, explainable business rule** here. Purchase quantity is confirmed (again) as a strong, cheap, interpretable early signal.

### Recommended Production Workflow
```
Score customers → Rank by repeat-purchase probability → Select Top-K
→ Target with retention campaign → Maintain control group → A/B test
→ Measure incremental repeat-purchase rate (not raw capture)
```

### Key Findings Summary
1. Repeat purchasing is rare (3.08% of modeling population).
2. First-order behavior does carry real, statistically validated signal (6 features significant post-FDR).
3. Purchase quantity is a particularly strong and cheap signal.
4. Higher first-order spend is NOT a positive predictor of repeat behavior.
5. Purchase timing (month, hour) matters; state and day-of-week do not.
6. Accuracy is a misleading model-selection metric in imbalanced problems.
7. CatBoost = best ranking model; XGBoost = best F1/classification model.
8. Overall predictive signal is real but weak (PR-AUC 0.0432 vs 3.08% base rate) — this is a **prioritization tool, not a deterministic predictor**.

### Limitations
Severe class imbalance; limited feature set (no category, payment type, channel, demographics, promo exposure); observational data (no causal claim); low absolute precision (~4%) at default threshold — many false positives if used as binary classifier; random (not temporal) train/test split — a production system should validate on a true future holdout; no probability calibration performed (Brier score / Platt scaling / isotonic regression recommended before treating scores as true probabilities).

---
## Notebook 07 — Executive Synthesis & Retention Strategy

**Purpose:** Notebook 07 is the **storytelling / executive synthesis notebook**. It introduces no new analysis — it takes the strongest findings from Notebooks 01–06 and assembles them into one coherent, end-to-end business narrative and decision framework.

### What It Covers

| Section | What It Covers |
|---|---|
| 1. LTV foundation | Reconstructed customer-level historical LTV and summarized customer value. |
| 2. LTV distribution | Visualized LTV distribution and customer-value concentration. |
| 3. Incentive economics | Visualized the economics of different incentive/conversion scenarios. |
| 4. Scenario limitations | Explicitly established that incentive ROI scenarios are assumptions, **not causal evidence**. |
| 5. A/B testing | Defined control vs. treatment, H₀/H₁, repeat conversion, incremental conversion lift, and business metrics. |
| 6. ML performance | Compared Logistic Regression, Random Forest, XGBoost, LightGBM, CatBoost, and AdaBoost using F1, PR-AUC, and ROC-AUC. |
| 7. Accuracy problem | Demonstrated why AdaBoost's **96.92% accuracy** is misleading when repeat customers are only **3.08%** of the population. |
| 8. Top-K targeting | Compared XGBoost and CatBoost for targeting the highest-propensity customers. |
| 9. CatBoost result | At Top-10%, CatBoost targeted **1,865 customers**, captured **89 repeat customers**, achieving **15.48% capture** and **1.548× lift**. |
| 10. Baseline comparison | Compared ML against the simple **first-order item-count** rule. |
| 11. ML vs. simple rule insight | Showed that first-order items achieved **1.461× lift** — very close to CatBoost's **1.548×**. |
| 12. Final ML recommendation | CatBoost = primary ranking model; XGBoost = alternative classification model; simple item-count rule = important benchmark. |
| 13. Full retention strategy | Connected **Identify → Segment → Rank → Target → Experiment → Measure → Improve** into a single operating loop. |
| 14. Core product insight | Established that this is a **retention prioritization system**, not a perfect repeat-purchase predictor. |
| 15. Evidence scorecard | Pulled the strongest numbers from the entire project into one summary. |
| 16. Final recommendations | Converted the analytical findings into concrete, actionable business decisions. |
| 17. Limitations | Covered observational data, class imbalance, weak predictive signal, LTV limitations, incentive assumptions, revenue vs. profit, and more. |
| 18. Future roadmap | Proposed uplift modeling, time-to-event modeling, incentive optimization, profit-based optimization, production deployment, and continuous experimentation. |

### The Overall Story of Notebook 07

The notebook is structured as a chain of business questions, each answered by a prior notebook:

| Question | Answer | Source |
|---|---|---|
| **Who are our customers?** | Quantified via historical LTV | Notebook 02 |
| **Who is worth retaining?** | High-value one-time customers | Notebook 02 / 03 |
| **What do we know about their behavior?** | First-order behavioral and category-level signals | Notebook 03 / 05 / 06 |
| **Can we intervene economically?** | Incentive scenario and ROI analysis | Notebook 03 / 04 |
| **Can we prove the intervention works?** | A/B testing framework | Notebook 04 |
| **Can we prioritize whom to target?** | ML-based ranking (CatBoost / XGBoost) | Notebook 06 |
| **Does sophisticated ML definitely beat simple rules?** | Not by much — first-order item count is a surprisingly strong baseline (1.461× vs. CatBoost's 1.548×) | Notebook 06 |
| **What should the business actually do?** | Target a limited high-potential segment, experiment with interventions, measure incremental impact, and scale only what works | Notebook 07 |

### Why This Notebook Matters

Notebook 07 is the **capstone** of the project. It doesn't add new numbers — it reframes every number from Notebooks 01–06 as part of a single, repeatable decision system:

## Cross-Notebook Business Recommendations

1. **Do not run universal checkout discounts.** Revenue and repeat-propensity are both highly concentrated; blanket incentives waste budget on customers unlikely to respond or of low economic value.
2. **Primary target: the 45,432 "high-value, one-time" customers** — 47.28% of the customer base, 75.18% of historical revenue, and (per Notebook 04) 7.21× more likely to repeat than low-value customers.
3. **Incentive sizing matters more than incentive presence.** Modeled ROI collapses from ~2,800% ($5 incentive) to 0% (~$147–149 incentive, the observed avg. second-order value). A $25 reference incentive was chosen as a strong, testable middle ground (~494% modeled ROI at 5% conversion).
4. **No single fixed campaign window works.** Repeat purchases cluster early (36.6% within 7 days) but also occur many months later (14–15% after 180+ days) — supports a multi-stage engagement strategy (early nudge → mid-term reminder → long-term reactivation).
5. **Category matters, but only 5 categories clear the bar** for statistically + commercially justified prioritization: bed_bath_table, computers_accessories, furniture_decor, fashion_bags_accessories, home_appliances (~22% of category revenue).
6. **Use ML for ranking, not classification.** CatBoost (best AUC/PR-AUC) and XGBoost (best F1) should score and rank customers for Top-K campaign selection; a fixed 0.5 threshold produces unusable precision.
7. **A simple "first-order item count" rule is a legitimate, cheap benchmark** — nearly matches ML lift at Top-10% (1.461× vs 1.548×) and should always be run alongside any ML system as a sanity check / fallback.
8. **Every revenue/ROI number in this project is a scenario estimate, not a forecast.** The single most repeated caveat across all 6 notebooks: historical association is not causation. **The mandatory next step is a controlled A/B test** (control vs. treatment tiers) measuring incremental lift over a control group, not raw conversion counts.

---

## Key Numbers Cheat Sheet

- Total customers (unique): **96,096** | Total historical revenue: **$16.01M**
- One-time customers: **93,099 (96.88%)** | Repeat customers: **2,997 (3.12%)**
- Avg LTV: one-time **$161.82** vs repeat **$314.99** (1.95× / 2.14× median)
- Top 20% of customers generate **53.77%** of revenue
- High-value threshold: **$319.57** → 9,611 customers (10%) generate **38.52%** of revenue
- **Target segment (high-value one-time): 45,432 customers, 47.28% of base, 75.18% of revenue**
- High-value repeat rate **5.48%** vs low-value **0.76%** → **7.21× lift**
- Avg 2nd-order value: **~$146.94–148.51**
- Reference incentive: **$25** → modeled ROI **~494%** at 5% conversion (2,272 conversions, ~$337K revenue, ~$281K net)
- Category repeat-item baseline: **6.72%**; 5 High-Priority categories = **21.98%** of category revenue
- ML repeat-prediction base rate: **3.08%**; best model (CatBoost) ROC-AUC **0.5770**, PR-AUC **0.0432**, Top-10% lift **1.548×**

---

## Limitations (Project-Wide)

- All findings are **observational**, not experimental — no causal claims are made anywhere in the project.
- Revenue ≠ profit: no cost of goods, margin, discount cost, or fulfillment cost is netted out anywhere except in the explicit ROI-vs-incentive-cost scenarios.
- Historical repeat rate and second-order value are backward-looking baselines, not guaranteed future outcomes.
- ML predictive signal, while statistically real, is weak in absolute terms (PR-AUC 0.0432) — suitable for **ranking/prioritization**, not deterministic prediction.
- Data anomalies (missing timestamps, financial discrepancies, chronological inconsistencies) were flagged, not removed — downstream models inherit this uncertainty.
- Geolocation data has known quality issues and should not be used for geospatial modeling without further validation.
- **The single overriding recommendation across every notebook: validate everything via a controlled A/B experiment before production deployment.**

---

## Tech Stack
* **Language:** Python
* **Data Manipulation:** Pandas, NumPy
* **Visualization:** Matplotlib, Seaborn
* **Statistical Analysis:** SciPy, Statsmodels — Mann–Whitney U test, Chi-square test, Benjamini–Hochberg FDR correction
* **Machine Learning:** Scikit-learn — Logistic Regression, Random Forest, AdaBoost; XGBoost 3.4.1; LightGBM 4.7.0; CatBoost 1.2.10
* **Dataset:** Olist Brazilian E-Commerce Public Dataset


