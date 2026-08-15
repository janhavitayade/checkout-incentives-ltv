# Checkout Incentives & LTV

## Project Overview

This project analyzes e-commerce customer and order data to investigate:

1. A/B Testing
2. Customer Lifetime Value (LTV) Modeling
3. Checkout Incentives
4. Product Sense

The project uses the Brazilian Olist e-commerce dataset and is designed to answer both
technical data-analysis questions and business-oriented questions around customers,
orders, revenue, retention, incentives, and purchasing behavior.

---

# 01 — Data Audit

Before performing analysis or modeling, a comprehensive data audit was conducted to
understand the structure, quality, completeness, consistency, and relationships between
the datasets.

The audit covered:

- Dataset structure and data types
- Missing values
- Duplicate records
- Invalid values
- Date/time consistency
- Order-status consistency
- Payment consistency
- Customer-order relationships
- Cross-table relationships
- Financial consistency
- Review chronology
- Geographic data quality

---

## Dataset Structure

The project contains the following datasets:

| Dataset | Purpose |
|---|---|
| Customers | Customer information and location |
| Geolocation | ZIP-code-level latitude, longitude and location information |
| Orders | Order lifecycle and timestamps |
| Order Items | Products purchased, sellers, prices and freight |
| Payments | Payment methods, installments and payment amounts |
| Reviews | Customer reviews and review scores |
| Products | Product characteristics |
| Sellers | Seller information and location |
| Category Translation | Portuguese → English product category mapping |

---

# Audit Findings

## 1. Data Types

| Dataset | Finding |
|---|---|
| Customers | Correct types: IDs as strings, ZIP code as integer |
| Geolocation | ZIP code as integer, latitude/longitude as numeric |
| Orders | IDs as strings; order timestamps converted to datetime |
| Order Items | IDs as strings/integer; price and freight as numeric |
| Payments | IDs as strings; payment values and installments numeric |
| Reviews | IDs as strings; review score numeric; review dates converted to datetime |
| Products | Product ID as string; product attributes numeric where applicable |
| Sellers | IDs as strings; ZIP code as integer |
| Category Translation | Category names stored as strings |

Date columns requiring analysis were converted from strings to datetime.

---

# 2. Missing Values

| Dataset | Column | Missing Values |
|---|---|---:|
| Orders | order_approved_at | 160 |
| Orders | order_delivered_carrier_date | 1,783 |
| Orders | order_delivered_customer_date | 2,965 |
| Reviews | review_comment_title | 87,656 |
| Reviews | review_comment_message | 58,247 |
| Products | product_category_name | 610 |
| Products | product_name_lenght | 610 |
| Products | product_description_lenght | 610 |
| Products | product_photos_qty | 610 |
| Products | product_weight_g | 2 |
| Products | product_length_cm | 2 |
| Products | product_height_cm | 2 |
| Products | product_width_cm | 2 |

Other datasets contained no missing values.

### Interpretation

Missing order timestamps are strongly related to the order lifecycle. For example,
orders that were not delivered may naturally lack a customer-delivery timestamp.

Review titles and messages contain substantial missingness because customers are not
required to provide written comments.

Product-level missing values are concentrated in a small number of products.

---

# 3. Order Status vs Missing Timestamps

| Order Status | Total | Approved Missing | Shipment Missing | Delivered Missing |
|---|---:|---:|---:|---:|
| approved | 2 | 0/2 | 2/2 | 2/2 |
| canceled | 625 | 141/625 | 550/625 | 619/625 |
| created | 5 | 5/5 | 5/5 | 5/5 |
| delivered | 96,478 | 14/96,478 | 2/96,478 | 8/96,478 |
| invoiced | 314 | 0/314 | 314/314 | 314/314 |
| processing | 301 | 0/301 | 301/301 | 301/301 |
| shipped | 1,107 | 0/1,107 | 0/1,107 | 1,107/1,107 |
| unavailable | 609 | 0/609 | 609/609 | 609/609 |

### Important Finding

There are **96,478 delivered orders**, but:

- 14 are missing approval timestamps
- 2 are missing shipment timestamps
- 8 are missing customer-delivery timestamps

These records should be treated as data-quality anomalies because a delivered order
would normally be expected to contain the relevant lifecycle timestamps.

---

# 4. Duplicate Records

| Dataset | Exact Duplicate Rows |
|---|---:|
| Customers | 0 |
| Geolocation | 261,831 |
| Orders | 0 |
| Order Items | 0 |
| Payments | 0 |
| Reviews | 0 |
| Products | 0 |
| Sellers | 0 |
| Category Translation | 0 |

The geolocation dataset contains many repeated records.

The dataset contains:

- 1,000,163 total geolocation rows
- 261,831 exact duplicate rows
- 19,015 unique ZIP prefixes
- 718,463 unique coordinate pairs

Geolocation duplicates should not automatically be treated as errors because multiple
records can legitimately refer to the same geographic location.

---

# 5. Geolocation Quality

| Metric | Result |
|---|---:|
| Unique ZIP prefixes | 19,015 |
| Unique coordinate pairs | 718,463 |
| ZIP prefixes with multiple latitude values | 17,781 |
| ZIP prefixes with multiple longitude values | 17,780 |

An additional issue was identified in the raw geolocation data: latitude/longitude
values appeared inconsistent with expected geographic coordinate ranges.

Therefore, the geolocation table should be treated carefully and should not be blindly
used for geographic modeling without further validation.

---

# 6. Order Status Distribution

| Order Status | Orders |
|---|---:|
| delivered | 96,478 |
| shipped | 1,107 |
| canceled | 625 |
| unavailable | 609 |
| invoiced | 314 |
| processing | 301 |
| created | 5 |
| approved | 2 |

The overwhelming majority of orders are delivered.

---

# 7. Payment Type Distribution

| Payment Type | Records |
|---|---:|
| credit_card | 76,795 |
| boleto | 19,784 |
| voucher | 5,775 |
| debit_card | 1,529 |
| not_defined | 3 |

Three payment records have `payment_type = not_defined`.

All three have:

- payment_value = 0
- payment_installments = 1

These records require investigation rather than automatic deletion.

---

# 8. Review Score Distribution

| Review Score | Reviews |
|---:|---:|
| 1 | 11,424 |
| 2 | 3,151 |
| 3 | 8,179 |
| 4 | 19,142 |
| 5 | 57,328 |

The observed review-score range is:

- Minimum = 1
- Maximum = 5
- Unique scores = 1, 2, 3, 4, 5

Therefore, **5 is the highest possible rating and 1 is the lowest** within this
dataset.

The distribution is heavily concentrated toward high scores, particularly score 5.

---

# 9. Customer Geographic Distribution

| State | Unique Customers | Orders | Sales Value | Unique Sellers |
|---|---:|---:|---:|---:|
| SP | 40,302 | 41,746 | 5,202,955.05 | 2,549 |
| RJ | 12,384 | 12,852 | 1,824,092.67 | 1,751 |
| MG | 11,259 | 11,635 | 1,585,308.03 | 1,664 |
| RS | 5,277 | 5,466 | 750,304.02 | 1,232 |
| PR | 4,882 | 5,045 | 683,083.76 | 1,232 |
| SC | 3,534 | 3,637 | 520,553.34 | 1,038 |
| BA | 3,277 | 3,380 | 511,349.99 | 967 |
| DF | 2,075 | 2,140 | 302,603.94 | 786 |
| ES | 1,964 | 2,033 | 275,037.31 | 738 |
| GO | 1,952 | 2,020 | 294,591.95 | 724 |
| PE | 1,609 | 1,652 | 262,788.03 | 638 |
| CE | 1,313 | 1,336 | 227,254.71 | 528 |
| PA | 949 | 975 | 178,947.81 | 465 |
| MT | 876 | 907 | 156,453.53 | 465 |
| MA | 726 | 747 | 119,648.22 | 375 |
| MS | 694 | 715 | 116,812.64 | 404 |
| PB | 519 | 536 | 115,268.08 | 303 |
| PI | 482 | 495 | 86,914.08 | 297 |
| RN | 474 | 485 | 83,034.98 | 275 |
| AL | 401 | 413 | 80,314.81 | 252 |
| SE | 342 | 350 | 58,920.85 | 212 |
| TO | 273 | 280 | 49,621.74 | 199 |
| RO | 240 | 253 | 46,140.64 | 182 |
| AM | 143 | 148 | 22,356.84 | 118 |
| AC | 77 | 81 | 15,982.95 | 71 |
| AP | 67 | 68 | 13,474.30 | 63 |
| RR | 45 | 46 | 7,829.43 | 39 |

SP, RJ and MG dominate both customer counts and order activity.

However, customer count alone does not prove demand. Orders and sales value were
therefore included to distinguish registered customers from actual purchasing
activity.

---

# 10. Customer Identity

| Metric | Result |
|---|---:|
| Customers rows | 99,441 |
| Unique customer_id | 99,441 |
| Unique customer_unique_id | 96,096 |
| Customers with multiple customer_id records | 2,997 |
| Maximum customer_id records for one customer_unique_id | 17 |

This distinction is important:

`customer_id` identifies a customer record associated with an order, whereas
`customer_unique_id` represents the underlying unique customer.

Therefore, `customer_unique_id` should be used when calculating customer-level
metrics such as repeat purchases and LTV.

---

# 11. Customer Repeat Purchase Behavior

| Orders per Customer | Customers |
|---:|---:|
| 1 | 93,099 |
| 2 | 2,745 |
| 3 | 203 |
| 4 | 30 |
| 5 | 8 |
| 6 | 6 |
| 7 | 3 |
| 9 | 1 |
| 17 | 1 |

Overall:

| Metric | Result |
|---|---:|
| One-order customers | 93,099 |
| Repeat customers | 2,997 |
| Maximum orders by one customer | 17 |

This is particularly important for the upcoming LTV analysis because the dataset contains
a large number of one-time customers and a smaller repeat-customer population.

---

# 12. Customer → Order Integrity

| Check | Result |
|---|---:|
| Orders without customer_id match | 0 |
| Customers without an order | 0 |
| Orders without customer_unique_id | 0 |

The customer-to-order relationship is complete.

---

# 13. Payment → Order Integrity

| Check | Result |
|---|---:|
| Payments without matching order | 0 |
| Orders without payment records | 1 |

Only one order lacks a corresponding payment record.

Order:

`30710`

The order is marked as delivered and contains order items, but no payment record
exists.

This should be treated as a missing/incomplete payment record rather than assuming that
the customer never paid.

---

# 14. Payment Records per Order

| Payment Records | Orders |
|---:|---:|
| 1 | 96,479 |
| 2 | 2,382 |
| 3 | 301 |
| 4 | 108 |
| 5 | 52 |
| 6 | 36 |
| 7 | 28 |
| 8 | 11 |
| 9 | 9 |
| 10 | 5 |
| 11 | 8 |
| 12 | 8 |
| 13 | 3 |
| 14 | 2 |
| 15 | 2 |
| 19 | 2 |
| 21 | 1 |
| 22 | 1 |
| 26 | 1 |
| 29 | 1 |

There are 2,961 orders with multiple payment records.

The maximum number of payment records for one order is 29.

Multiple payment records are possible because an order may involve multiple payment
transactions, vouchers or payment methods.

---

# 15. Order Items per Order

| Items | Orders |
|---:|---:|
| 1 | 88,863 |
| 2 | 7,516 |
| 3 | 1,322 |
| 4 | 505 |
| 5 | 204 |
| 6 | 198 |
| 7 | 22 |
| 8 | 8 |
| 9 | 3 |
| 10 | 8 |
| 11 | 4 |
| 12 | 5 |
| 13 | 1 |
| 14 | 2 |
| 15 | 2 |
| 20 | 2 |
| 21 | 1 |

There are 9,803 orders containing multiple items.

The maximum number of items in one order is 21.

---

# 16. Order Item Financial Data

| Metric | Price | Freight |
|---|---:|---:|
| Mean | 120.65 | 19.99 |
| Median | 74.99 | 16.26 |
| Minimum | 0.85 | 0.00 |
| Maximum | 6,735.00 | 409.68 |

No negative prices or negative freight values were found.

There were 383 records with zero freight value.

---

# 17. Payment Financial Data

| Metric | Payment Value | Installments |
|---|---:|---:|
| Mean | 154.10 | 2.85 |
| Median | 100.00 | 1 |
| Minimum | 0.00 | 0 |
| Maximum | 13,664.08 | 24 |

No negative payment values were found.

There were:

- 9 zero-payment records
- 2 records with invalid zero installments

These records were flagged for further investigation.

---

# 18. Chronological Consistency

| Check | Records |
|---|---:|
| Approval before purchase | 0 |
| Shipment before purchase | 166 |
| Delivery before purchase | 0 |
| Delivery before shipment | 23 |

The problematic records are relatively small compared with the full dataset:

- Shipment before purchase: 166 records
- Delivery before shipment: 23 records

These should be flagged as chronological anomalies rather than automatically removed.

---

# 19. Review Chronology

Reviews were checked against the order lifecycle.

| Check | Records |
|---|---:|
| Review before purchase | 74 |
| Review before approval | 33 |
| Review before shipment | 296 |
| Review before delivery | 8,320 |

The review chronology requires caution because review creation dates do not necessarily
represent the exact moment the customer received the product.

Therefore, these records should be investigated before being classified as invalid.

---

# 20. Example Chronological Anomaly

Order `73222` was identified with:

- Purchase: 2017-09-29
- Approval: 2017-09-29
- Shipment: missing
- Delivery: 2017-11-20
- Estimated delivery: 2017-11-14

The actual delivery occurred after the estimated delivery date.

This is a useful example of why actual delivery performance should be compared with
estimated delivery rather than assuming the estimated date represents the actual
delivery.

---

# 21. Order → Order Item Coverage

| Check | Result |
|---|---:|
| Orders without order items | 775 |
| Order items with unknown order | 0 |

This means every order-item record belongs to a valid order, but 775 orders have no
corresponding item record.

---

# 22. Order → Payment Coverage

| Check | Result |
|---|---:|
| Orders without payments | 1 |
| Payments with unknown order | 0 |

Payment records are therefore structurally valid, with one order lacking payment
information.

---

# 23. Order → Review Coverage

| Check | Result |
|---|---:|
| Orders without reviews | 768 |
| Reviews with unknown order | 0 |

Not every order has a review, but every review belongs to a valid order.

This is expected because leaving a review is optional.

---

# 24. Financial Consistency

Order-level item totals were compared with payment totals.

For orders containing both item and payment records:

| Metric | Result |
|---|---:|
| Orders with item records | 98,666 |
| Orders with payment records | 99,440 |
| Orders with both | 98,665 |
| Exact matches | 98,284 |
| Differences > $0.01 | 381 |
| Maximum absolute difference | $182.81 |
| Mean absolute difference | $0.033 |
| Median absolute difference | $0.00 |

Among the 381 discrepancies:

| Relationship | Orders |
|---|---:|
| Payment > item total | 291 |
| Payment < item total | 90 |

---

# 25. Financial Discrepancy Range

Only discrepancies greater than $0.01 were considered.

| Metric | Difference |
|---|---:|
| Minimum absolute difference | $0.01 |
| Maximum absolute difference | $182.81 |

The 381 discrepancies should be flagged rather than automatically removed.

Possible explanations include additional charges, discounts, vouchers, adjustments,
rounding, or differences in how item and payment totals were recorded.

However, because the other 98,284 orders match exactly, the discrepancies deserve
additional investigation rather than being dismissed as universally applicable
platform charges.

---

# 26. Overall Data Integrity

| Area | Finding | Action |
|---|---|---|
| Duplicate records | Geolocation contains many duplicates | Investigate before use |
| Missing order timestamps | Present across several statuses | Treat according to order lifecycle |
| Delivered orders with missing timestamps | 24 total across relevant timestamp fields | Flag for investigation |
| Invalid chronology | 166 shipment-before-purchase records | Flag |
| Invalid chronology | 23 delivery-before-shipment records | Flag |
| Missing payment | 1 order | Investigate |
| Missing order items | 775 orders | Investigate before item-level analysis |
| Missing reviews | 768 orders | Expected to some extent |
| Payment discrepancies | 381 orders | Investigate |
| Zero payments | 9 records | Investigate |
| Invalid installments | 2 records | Investigate |
| Customer-order matching | Complete | No action required |
| Payment-order matching | Complete except one order | Flag one order |
| Review-order matching | Complete | No action required |

---

# Audit Conclusion

The dataset is structurally usable for further analysis, but several data-quality issues
were identified.

The most important findings are:

1. Customer-to-order relationships are complete.
2. Payment-to-order relationships are almost complete, with one missing payment record.
3. Review and order-item records do not exist for every order.
4. A small number of order timestamps violate the expected chronological sequence.
5. Some delivered orders have missing lifecycle timestamps.
6. 381 orders have differences between item-derived totals and payment totals.
7. Customer-level analysis should use `customer_unique_id` rather than `customer_id`.
8. The customer base is dominated by one-time purchasers.
9. SP, RJ and MG dominate customer and order activity.
10. Geolocation contains substantial duplication and requires additional validation.
11. Several anomalous records should be flagged rather than blindly deleted.

No records were automatically deleted solely because they were anomalous. The purpose of
the audit is to identify potential data-quality problems while preserving the original
data for downstream analysis.

These findings will be considered when preparing the data for:

- LTV modeling
- A/B testing
- Checkout incentive analysis
- Customer segmentation
- Product and business analysis


# 02 — Customer Lifetime Value (LTV) Modeling

## 1. Objective

The purpose of this notebook is to quantify the **economic value of customers** and identify customer segments that are most relevant for targeted checkout incentives.

The analysis answers:

* How much revenue does the average customer generate?
* How many customers make only one purchase versus repeat purchases?
* Do repeat customers generate substantially higher LTV?
* How concentrated is revenue among high-value customers?
* What does customer lifetime look like?
* How quickly do customers make a second purchase?
* Which customer segments should be prioritized for incentives?
* Are there high-value customers who have purchased only once and therefore represent potential repeat-purchase opportunities?

The notebook converts transaction-level order/payment data into a **customer-level LTV dataset** and then performs segmentation and revenue-concentration analysis.

---

# 2. Data Used

The analysis is based on the Olist e-commerce transactional dataset.

The main information required for LTV modeling is:

| Data          | Purpose                                     |
| ------------- | ------------------------------------------- |
| Customer data | Identify unique customers                   |
| Orders data   | Determine purchases and purchase timestamps |
| Payments data | Calculate customer revenue/spend            |

The analysis uses `customer_unique_id` rather than `customer_id`.

### Why `customer_unique_id`?

`customer_id` represents an individual customer record associated with an order, whereas `customer_unique_id` identifies the underlying customer across the dataset.

Therefore:

> **LTV must be calculated at the `customer_unique_id` level**, otherwise repeat purchases by the same customer could incorrectly appear as separate customers.

---

# 3. Customer-Level LTV Dataset

Each customer was aggregated into a single record containing:

| Column               | Meaning                                       |
| -------------------- | --------------------------------------------- |
| `customer_unique_id` | Unique customer identifier                    |
| `orders`             | Number of distinct orders                     |
| `total_spend`        | Total payment value generated by the customer |
| `first_purchase`     | Timestamp of first purchase                   |
| `last_purchase`      | Timestamp of last purchase                    |
| `lifetime_days`      | Days between first and last purchase          |

### LTV definition

For this project, historical customer LTV is defined as:

> **LTV = total amount spent by a customer across all recorded orders**

This is an observed/historical LTV measure rather than a future-value prediction.

### Why use total payment value?

Payment value represents the actual monetary value associated with the customer's purchases and therefore provides the most direct measure of customer revenue contribution available in the dataset.

---

# 4. Overall Customer LTV

| Metric          |            Result |
| --------------- | ----------------: |
| Total customers |        **96,096** |
| Total revenue   | **16,008,872.12** |
| Average LTV     |        **166.59** |
| Median LTV      |        **108.00** |
| Minimum LTV     |          **0.00** |
| Maximum LTV     |     **13,664.08** |

### Interpretation

The average customer generated approximately **166.59** in historical revenue.

However, the median is only **108.00**, substantially below the mean.

This indicates that customer spending is **right-skewed**: a relatively small number of customers spend substantially more than the typical customer.

The maximum observed LTV of **13,664.08** further confirms the presence of extremely high-spending customers.

### Why report both mean and median?

The mean is useful for estimating aggregate economic value, while the median better represents the typical customer because it is less affected by extreme spending values.

---

# 5. One-Time vs Repeat Customers

Customers were classified as:

* **One-time:** exactly 1 order
* **Repeat:** more than 1 order

| Customer type |  Customers | % of customers | Avg orders |    Avg LTV | Median LTV |
| ------------- | ---------: | -------------: | ---------: | ---------: | ---------: |
| One-time      | **93,099** |     **96.88%** |      1.000 | **161.82** | **105.70** |
| Repeat        |  **2,997** |      **3.12%** |      2.116 | **314.99** | **225.84** |

### Key finding

Only **3.12%** of customers made more than one purchase.

However, repeat customers have:

* approximately **1.95× higher average LTV**
* approximately **2.14× higher median LTV**

than one-time customers.

### Why is this important?

This is one of the most important findings for the incentive problem.

A customer who makes a second purchase is considerably more valuable on average than a customer who purchases only once.

Therefore, encouraging selected one-time customers to return can potentially create significant incremental value.

---

# 6. Revenue Contribution by Customer Type

| Customer type | Customers | Customer % |       Revenue |  Revenue % |
| ------------- | --------: | ---------: | ------------: | ---------: |
| One-time      |    93,099 | **96.88%** | 15,064,849.41 | **94.10%** |
| Repeat        |     2,997 |  **3.12%** |    944,022.71 |  **5.90%** |

### Interpretation

The majority of historical revenue comes from one-time customers simply because they represent almost the entire customer base.

However, this should **not** be interpreted as one-time customers being more valuable individually.

At the individual-customer level:

**Repeat LTV = 314.99**

versus

**One-time LTV = 161.82**

Therefore:

> Repeat customers are individually more valuable, even though they represent only a small proportion of the customer base.

This distinction is important when designing targeted incentives.

---

# 7. LTV Distribution

Customers were divided into LTV bands.

| LTV band  | Customers | % of customers |
| --------- | --------: | -------------: |
| <50       |    15,849 |     **16.49%** |
| 50–100    |    28,546 |     **29.71%** |
| 100–250   |    37,454 |     **38.98%** |
| 250–500   |     9,758 |     **10.15%** |
| 500–1000  |     3,271 |      **3.40%** |
| 1000–2000 |       993 |      **1.03%** |
| 2000+     |       225 |      **0.23%** |

### Interpretation

The largest customer group falls into the **100–250 LTV range**, representing **38.98%** of customers.

Approximately:

* **46.20%** have LTV below 100.
* **38.98%** have LTV between 100 and 250.
* **10.15%** have LTV between 250 and 500.
* Only **4.67%** have LTV above 500.

This demonstrates that extremely high-value customers are relatively rare.

---

# 8. Revenue Concentration

Revenue concentration was measured by calculating the percentage of total revenue generated by the highest-value customers.

| Customer segment | Revenue contribution |
| ---------------- | -------------------: |
| Top 1%           |           **10.48%** |
| Top 5%           |           **27.04%** |
| Top 10%          |           **38.51%** |
| Top 20%          |           **53.77%** |

### Interpretation

The top **20% of customers generate 53.77% of revenue**.

The top **10% generate 38.51%**.

The top **5% generate 27.04%**.

This demonstrates strong **revenue concentration**.

### Business implication

Not every customer should necessarily receive the same incentive.

A blanket incentive strategy could waste budget on customers with little incremental value.

A more efficient strategy is to identify customers according to their expected economic value and likelihood of responding to an incentive.

---

# 9. LTV by Number of Orders

| Orders | Customers |  Avg LTV | Median LTV | Total revenue |
| -----: | --------: | -------: | ---------: | ------------: |
|      1 |    93,099 |   161.82 |     105.70 | 15,064,849.41 |
|      2 |     2,745 |   294.85 |     217.74 |    809,355.39 |
|      3 |       203 |   473.39 |     342.48 |     96,097.83 |
|      4 |        30 |   779.13 |     537.72 |     23,373.88 |
|      5 |         8 |   759.66 |     674.66 |      6,077.25 |
|      6 |         6 |   696.25 |     743.63 |      4,177.51 |
|      7 |         3 |   946.85 |     959.01 |      2,840.56 |
|      9 |         1 | 1,172.66 |   1,172.66 |      1,172.66 |
|     17 |         1 |   927.63 |     927.63 |        927.63 |

### Interpretation

The general pattern is clear:

> **More purchases are associated with higher customer LTV.**

For example:

* 1-order customers → **161.82 average LTV**
* 2-order customers → **294.85**
* 3-order customers → **473.39**
* 4-order customers → **779.13**

The small irregularities at very high order counts are caused by extremely small sample sizes.

For example, only **1 customer** made 17 orders.

Therefore, those extreme groups should not be treated as statistically representative.

---

# 10. High-Value Customer Threshold

A high-value threshold was established at:

> **LTV = 319.57**

Customers above this threshold were classified as **high-value**.

| Segment    |  Customers |    Avg LTV | Median LTV |          Revenue |  Revenue % |
| ---------- | ---------: | ---------: | ---------: | ---------------: | ---------: |
| High-value |  **9,611** | **641.55** | **476.14** | **6,165,903.93** | **38.52%** |
| Standard   | **86,485** | **113.81** |  **97.77** | **9,842,968.19** | **61.48%** |

### Interpretation

Only approximately **10% of customers** are classified as high-value.

Yet they generate approximately **38.52% of total revenue**.

This confirms that customer value is highly concentrated.

### Business implication

High-value customers deserve differentiated treatment because losing or successfully retaining one of these customers can have a much larger financial impact than doing the same with a low-value customer.

---

# 11. Customer Lifetime

Customer lifetime was calculated as:

> **last purchase timestamp − first purchase timestamp**

| Customer type | Customers |   Avg lifetime | Median lifetime |
| ------------- | --------: | -------------: | --------------: |
| One-time      |    93,099 |     **0 days** |      **0 days** |
| Repeat        |     2,997 | **87.31 days** |  **33.66 days** |

### Why do one-time customers have 0 lifetime?

For a one-time customer:

`first_purchase = last_purchase`

Therefore:

`lifetime_days = 0`

This is expected behavior and is **not missing data**.

### Interpretation

Repeat customers have substantially longer observed customer lifetimes.

The median repeat-customer lifetime is approximately **33.66 days**, while the mean is **87.31 days**.

The difference between mean and median indicates that some repeat customers remain active for substantially longer periods.

---

# 12. Time to Second Purchase

For repeat customers, the time between the first and second purchase was calculated.

| Metric                          |          Result |
| ------------------------------- | --------------: |
| Customers with repeat purchase  |       **2,997** |
| Average days to second purchase |  **80.35 days** |
| Median days to second purchase  |  **27.92 days** |
| Minimum                         |      **0 days** |
| Maximum                         | **608.98 days** |

### Interpretation

The median customer who makes a second purchase does so in approximately **28 days**.

However, the average is much higher at **80.35 days**.

This indicates a strongly right-skewed distribution: some customers return very quickly, while others return many months later.

---

# 13. Second-Purchase Timing Distribution

| Time to second purchase | Customers | Percentage |
| ----------------------- | --------: | ---------: |
| 1–7 days                | **1,097** | **36.60%** |
| 8–14 days               |       166 |  **5.54%** |
| 15–30 days              |       267 |  **8.91%** |
| 31–60 days              |       320 | **10.68%** |
| 61–90 days              |       208 |  **6.94%** |
| 91–180 days             |       424 | **14.15%** |
| 181–365 days            |       427 | **14.25%** |
| 365+ days               |        88 |  **2.94%** |

### Key finding

**36.60% of repeat customers made their second purchase within 7 days.**

Approximately:

* **51.1%** returned within 30 days.
* **61.8%** returned within 60 days.
* **68.7%** returned within 90 days.
* **82.8%** returned within 180 days.
* **97.1%** returned within one year.

### Business implication

The first few weeks after purchase represent an important potential intervention period.

However, the existence of customers returning months later means that a single fixed incentive window would not capture every repeat-purchase opportunity.

---

# 14. Incentive Segmentation

Customers were divided using two dimensions:

1. Customer value — high-value vs low-value
2. Purchase behavior — one-time vs repeat

This produced four actionable segments.

| Segment             |  Customers | Customer % | Avg orders |    Avg LTV | Median LTV |           Revenue |  Revenue % |
| ------------------- | ---------: | ---------: | ---------: | ---------: | ---------: | ----------------: | ---------: |
| High-value one-time | **45,432** | **47.28%** |       1.00 | **264.91** |     180.21 | **12,035,546.53** | **75.18%** |
| High-value repeat   |  **2,634** |  **2.74%** |      2.129 | **347.05** |     251.98 |    **914,135.72** |  **5.71%** |
| Low-value one-time  | **47,667** | **49.60%** |       1.00 |  **63.55** |      63.00 |  **3,029,302.88** | **18.92%** |
| Low-value repeat    |    **363** |  **0.38%** |      2.022 |  **82.33** |      85.07 |     **29,886.99** |  **0.19%** |

---

# 15. Most Important Segmentation Finding

The **high-value one-time** segment is particularly important.

It contains:

* **45,432 customers**
* **47.28% of the customer base**
* **75.18% of total revenue**
* Average LTV = **264.91**

These customers have already demonstrated substantial monetary value but have made only one purchase.

Therefore, they represent a potentially valuable **conversion-to-repeat** segment.

### Why?

They satisfy two desirable conditions:

**High historical value + no repeat purchase yet**

This makes them potentially more attractive incentive targets than low-value one-time customers.

---

# 16. Low-Value Customers

The low-value one-time segment contains:

**47,667 customers**

and contributes only:

**18.92% of revenue.**

Their average LTV is only:

**63.55**

Therefore, spending large incentive budgets on this group could produce poor ROI unless there is evidence that incentives significantly increase their probability of repurchasing.

---

# 17. High-Value Repeat Customers

High-value repeat customers represent:

**2,634 customers (2.74% of the customer base)**

and generate:

**5.71% of total revenue.**

Their average LTV is:

**347.05**

These customers have already demonstrated repeat behavior.

Therefore, their incentive strategy should potentially focus more on **retention** rather than simply trying to induce a first repeat purchase.

---

# 18. Low-Value Repeat Customers

The low-value repeat segment contains only:

**363 customers (0.38%)**

and contributes:

**0.19% of total revenue.**

Their average LTV is:

**82.33**

This is a relatively low-priority segment for expensive incentives because its historical revenue contribution is extremely small.

---

# 19. Highest-Value Customers

The top customer records were inspected to identify extreme-value customers.

The largest observed customer had:

**LTV = 13,664.08**

Other extremely high-value customers included customers with LTVs above:

* 9,500
* 7,500
* 7,000
* 6,900
* 6,700
* 6,000
* 4,800
* 4,500

### Interpretation

The extreme values confirm that the customer-value distribution has a long right tail.

These customers should not automatically be removed as outliers because they represent genuine revenue generated by the business.

Instead, they demonstrate why median-based statistics and segmentation are useful alongside averages.

---

# 20. Important Data Quality/Interpretation Notes

| Observation                                         | Interpretation                                                                            |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| One-time lifetime = 0                               | Expected because first and last purchases are identical                                   |
| Minimum LTV = 0                                     | Can occur because payment/order aggregation includes zero-value transactions              |
| Very high LTV values                                | Genuine high-spend observations; should not automatically be deleted                      |
| Mean LTV > median LTV                               | Indicates right-skewed customer spending                                                  |
| Mean repeat lifetime > median                       | Indicates a long right tail in customer retention                                         |
| Very high order-count groups have tiny sample sizes | Their averages should not be generalized                                                  |
| Repeat customers are few                            | Repeat behavior is rare in the observed dataset                                           |
| Repeat customers have higher LTV                    | Strong evidence of association, not proof that repeat purchasing itself causes higher LTV |

---

# 21. Core Business Findings

| Finding                                                   | Evidence                          | Business meaning                                                               |
| --------------------------------------------------------- | --------------------------------- | ------------------------------------------------------------------------------ |
| Customer spending is highly skewed                        | Mean LTV 166.59 vs median 108.00  | A minority of customers generate disproportionately high value                 |
| Repeat customers are more valuable                        | Avg LTV 314.99 vs 161.82          | Repeat purchase behavior is strongly associated with higher customer value     |
| Repeat customers are rare                                 | 3.12% of customers                | Increasing repeat-purchase rate represents a potentially important opportunity |
| Revenue is concentrated                                   | Top 20% → 53.77% revenue          | Customer targeting should be value-aware                                       |
| High-value customers are a minority                       | 9,611 customers                   | High-value segmentation can reduce inefficient blanket incentives              |
| High-value customers generate substantial revenue         | 38.52% of revenue                 | Retention of high-value customers is economically important                    |
| High-value one-time customers are particularly attractive | 47.28% customers → 75.18% revenue | Potentially valuable population for repeat-purchase conversion                 |
| Low-value one-time customers have low LTV                 | Avg LTV 63.55                     | Expensive incentives may have limited economic justification                   |
| Many repeat purchases happen quickly                      | 36.60% within 7 days              | Early post-purchase period may be an important intervention window             |
| Repeat purchases also occur much later                    | 14.25% at 181–365 days            | Incentive timing should not assume every customer behaves on the same schedule |

---

# 22. What This Notebook Establishes

The analysis establishes four important facts for the next stage of the project:

### 1. Customer value is heterogeneous

Customers do not contribute equal amounts of revenue.

A small fraction generates a disproportionately large share of revenue.

### 2. Repeat purchasing is strongly associated with higher LTV

Repeat customers have approximately twice the average historical LTV of one-time customers.

### 3. One-time customers cannot be treated equally

A customer with historical LTV of 50 and a customer with historical LTV of 300 should not necessarily receive the same incentive.

### 4. Incentives should be targeted

The analysis supports moving away from:

> "Give every customer a discount."

toward:

> "Identify customers where an incentive is likely to generate enough incremental value to justify its cost."

---

# 23. Limitations of This LTV Analysis

This notebook calculates **historical LTV**, not predicted future LTV.

Therefore:

* It does not predict whether a customer will purchase again.
* It does not estimate the probability of responding to an incentive.
* It does not calculate incremental revenue caused by an incentive.
* It does not establish causality between incentives and repeat purchases.
* It does not account for profit margin unless margin data is introduced.
* It does not subtract incentive/discount cost.
* It does not explicitly model customer churn probability.

These limitations are intentional.

The purpose of this notebook is to establish the **economic/customer-value foundation** for subsequent modeling.

---

# 24. Final Conclusion

The LTV analysis reveals a highly uneven customer-value distribution.

There are **96,096 customers** generating approximately **16.01 million** in historical revenue. Although **96.88%** of customers are one-time purchasers, repeat customers have substantially higher individual LTV.

Only **3.12%** of customers are repeat purchasers, yet their average LTV is approximately **315**, compared with approximately **162** for one-time customers.

Revenue is also highly concentrated: the top **10% of customers generate 38.51% of revenue**, while the top **20% generate 53.77%**.

The most strategically important segment is the **high-value one-time customer group**. These customers represent **47.28% of the customer base but account for 75.18% of revenue**, while having made only one purchase.

This makes them a particularly important population for the next stage of the project: determining **which customers should receive checkout incentives and whether the expected incremental value can justify the incentive cost**.

Therefore, `02_ltv_modeling` provides the customer-value foundation required for the subsequent incentive-targeting/modeling stage.
