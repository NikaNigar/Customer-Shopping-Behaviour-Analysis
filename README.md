# Customer-Shopping-Behaviour-Analysis
An end-to-end data analytics project that analyses 3,900 customer purchases with Python, SQL and Power BI. The goal is to understand who the customers are, what they buy and how to grow revenue.
# 🛒 Customer Shopping Behaviour Analysis

![Python](https://img.shields.io/badge/Python-Pandas%20%7C%20Matplotlib-3776AB?logo=python&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL-SQL%20Server-CC2927?logo=microsoftsqlserver&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-Dashboard-F2C811?logo=powerbi&logoColor=black)

An end-to-end data analytics project that analyses **3,900 customer purchases** with **Python, SQL and Power BI**. The goal is to understand who the customers are, what they buy and how to grow revenue.

![Power BI Dashboard](images/dashboard.png)

---

## 📌 1. Project Overview

A retail company wants to understand how its customers shop: which products sell best, which customer groups spend the most, and how discounts, subscriptions, shipping and seasons affect buying.

**Business questions**
- Who are our customers (age, gender, location) and how much do they spend?
- Which categories and products bring in the most revenue and the best reviews?
- Do discounts, subscriptions and faster shipping make customers spend more?
- How many customers are new, returning or loyal?

**Workflow**

```
CSV data  →  Python (cleaning + EDA)  →  SQL Server (business questions)  →  Power BI (dashboard)  →  Recommendations
```

### Headline numbers

| Customers | Total revenue | Avg purchase | Avg rating | Avg previous purchases | Avg days between purchases |
|:-:|:-:|:-:|:-:|:-:|:-:|
| **3,900** | **$233,081** | **$59.76** | **3.75 / 5** | **25.35** | **89 days** |

---

## 📁 2. Dataset Summary

- **Rows:** 3,900 (one purchase per customer) · **Columns:** 18
- **Missing values:** 37 in `Review Rating` (under 1%), filled with the median rating of each category
- **Duplicate column:** `Promo Code Used` had exactly the same values as `Discount Applied`, so it was removed

| Group | Columns |
|---|---|
| Customer | Customer ID, Age, Gender, Location, Subscription Status |
| Purchase | Item Purchased, Category, Purchase Amount (USD), Size, Color, Season |
| Behaviour | Review Rating, Shipping Type, Discount Applied, Promo Code Used, Previous Purchases, Payment Method, Frequency of Purchases |

**New columns created**

| Column | Description |
|---|---|
| `age_group` | Age split into 4 equal groups: young-adult, adult, middle_aged, senior |
| `purchase_frequency_days` | Purchase frequency converted into days (Weekly = 7 … Annually = 365) |

---

## 🐍 3. Exploratory Data Analysis (Python)

```python
import pandas as pd

df = pd.read_csv('customer_shopping_behavior.csv')

# Fill missing ratings with the median of each category
df['Review Rating'] = df.groupby('Category')['Review Rating'] \
                        .transform(lambda x: x.fillna(x.median()))

# Column names to snake_case
df.columns = df.columns.str.lower().str.replace(' ', '_')
df = df.rename(columns={'purchase_amount_(usd)': 'purchase_amount'})

# Age groups
df['age_group'] = pd.qcut(df['age'], q=4,
                          labels=['young-adult', 'adult', 'middle_aged', 'senior'])

# Purchase frequency in days
freq = {'Weekly': 7, 'Bi-Weekly': 14, 'Fortnightly': 14, 'Monthly': 30,
        'Quarterly': 90, 'Every 3 Months': 90, 'Annually': 365}
df['purchase_frequency_days'] = df['frequency_of_purchases'].map(freq)

# Remove duplicate column
df = df.drop('promo_code_used', axis=1)

# Load into SQL Server
df.to_sql('customer', engine, if_exists='replace', index=False)
```

**Key EDA findings**
- 👥 **68% of customers are male**, and they bring in 68% of revenue ($157.9K vs $75.2K).
- ⭐ **Only 27% are subscribers, and all 1,053 of them are male.** No female customer is subscribed.
- 👕 **Clothing ($104.3K) and Accessories ($74.2K) make up 77% of revenue.** Outerwear brings in only 8%.
- 🍂 **Fall is the best season** ($60.0K) and **Summer the weakest** ($55.8K).
- 🚚 2-Day, Express and Free shipping customers spend about **$2 more per order** than Standard customers.
- 🏷️ Discounts **do not increase order size** ($59.28 with a discount vs $60.13 without).

---

## 🗄️ 4. SQL Analysis (SQL Server)

| # | Business question | Key result |
|:-:|---|---|
| 1 | Revenue by gender | Male $157,890 · Female $75,191 |
| 2 | Discount users who spent above average | 839 customers |
| 3 | Top 5 products by average rating | Gloves 3.86 · Sandals 3.84 · Boots 3.82 |
| 4 | Express vs Standard shipping | $60 vs $58 average purchase |
| 5 | Subscribers vs non-subscribers | Same average spend ($59) |
| 6 | Top 5 products bought with a discount | Hat 50% · Coat 49% · Sneakers 49% |
| 7 | Customer segments | New 155 · Returning 629 · Loyal 3,116 |
| 8 | Top 3 products per category | Jewelry, Blouse, Sandals, Jacket lead their categories |
| 9 | Repeat buyers (5+ purchases) by subscription | Yes 980 · No 2,583 |
| 10 | Revenue by age group | Young-adult $62,143 is highest |

<details>
<summary><b>Example queries</b></summary>

```sql
-- Q7: Customer segments
WITH customer_type AS (
    SELECT customer_id,
           CASE WHEN previous_purchases <= 2  THEN 'New'
                WHEN previous_purchases <= 10 THEN 'Returning'
                ELSE 'Loyal' END AS customer_segment
    FROM customer
)
SELECT customer_segment, COUNT(*) AS number_of_customers
FROM customer_type
GROUP BY customer_segment;

-- Q8: Top 3 products in each category
WITH item_counts AS (
    SELECT category, item_purchased,
           COUNT(customer_id) AS total_orders,
           ROW_NUMBER() OVER (PARTITION BY category
                              ORDER BY COUNT(customer_id) DESC) AS dr
    FROM customer
    GROUP BY category, item_purchased
)
SELECT dr, category, item_purchased, total_orders
FROM item_counts
WHERE dr <= 3;

-- Q6: Products most often bought with a discount
SELECT TOP 5 item_purchased,
       SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) * 100 / COUNT(*) AS discount_rate
FROM customer
GROUP BY item_purchased
ORDER BY discount_rate DESC;
```
</details>

All 10 queries are in [`sql/customer_behavior_queries.sql`](sql/customer_behavior_queries.sql).

---

## 📊 5. Power BI Dashboard

An interactive dashboard with filters for **Category, Location, Age Group and Payment Method**.

- **KPI cards:** Avg Review Rating, Avg Previous Purchases, Avg Purchase Amount, Avg Purchase Days, Number of Customers, Total Purchase Amount
- **Visuals:** revenue by category, customers by age group, gender donut, US map, revenue by season, category treemap (drill-down), decomposition tree (category → age group → colour)

```DAX
Total Purchase Amount  = SUM(customer[purchase_amount])
Avg Purchase Amount    = AVERAGE(customer[purchase_amount])
Avg Review Rating      = AVERAGE(customer[review_rating])
Avg Previous Purchases = AVERAGE(customer[previous_purchases])
Avg Purchase Days      = AVERAGE(customer[purchase_frequency_days])
Number of Customers    = DISTINCTCOUNT(customer[customer_id])
```

---

## 💡 6. Business Recommendations

1. **Open the subscription to women.** None of the 1,248 female customers subscribe. Run a targeted campaign and add real member benefits (free Express shipping, early access, member prices).
2. **Convert loyal non-subscribers.** 2,583 customers with 5+ purchases are not subscribed, which makes them the easiest group to convert.
3. **Use discounts smarter.** Replace blanket discounts with "spend $80, get 10% off" offers, and reduce discounting on Hats, Coats and Sneakers.
4. **Focus on hero products.** Keep Jewelry, Blouse, Pants, Sandals and Jacket always in stock, bundle them with high-rated items, and grow Outerwear in Fall/Winter.
5. **Promote faster shipping and fix the Summer dip.** Offer free Express above a minimum order, and plan a summer campaign.
6. **Attract new customers and improve ratings.** Only 4% of customers are new, so add referral and first-purchase offers, and act on feedback for low-rated items.

---

## 📂 Project Structure

```
customer-shopping-behaviour-analysis/
├── data/
│   └── customer_shopping_behavior.csv
├── python/
│   └── customer_behavior_analysis.ipynb
├── sql/
│   └── customer_behavior_queries.sql
├── powerbi/
│   └── customer_behavior_dashboard.pbix
├── images/
│   └── dashboard.png
├── report/
│   └── Customer_Shopping_Behaviour_Analysis_Report.pdf
└── README.md
```

## ▶️ How to Run

1. Clone the repository: `git clone https://github.com/<your-username>/customer-shopping-behaviour-analysis.git`
2. Install the Python libraries: `pip install pandas matplotlib seaborn sqlalchemy pyodbc`
3. Run the notebook in `python/` to clean the data and load it into SQL Server.
4. Run the queries in `sql/customer_behavior_queries.sql`.
5. Open the `.pbix` file in Power BI Desktop and refresh the data.

---

## 👩‍💻 Author

**Nigar Shahmuradova** · Data Analyst · Microsoft Certified: Power BI Data Analyst Associate (PL-300)

⭐ If you found this project useful, please give it a star!
