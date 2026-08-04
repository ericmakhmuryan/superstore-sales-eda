# Superstore Sales EDA & PostgreSQL Analysis

In this project, I took a look at the Superstore Sales dataset to practice my data cleaning, visualization, and SQL skills. My main goal was to figure out what's actually driving sales, spot any weird outliers, check if discounts are actually helping or hurting profit, and segment customers into different value tiers using PostgreSQL.

## Tools & Libraries
* **Python**: `pandas` and `numpy` for data manipulation and math.
* **Visualization**: `seaborn` and `matplotlib` to plot charts.
* **Database**: `PostgreSQL` (connected using `sqlalchemy` and `psycopg2` in Python).
* **Environment**: Google Colab & DBeaver.

---

##What I Found Out (Key Takeaways)

* **Mean vs. Median is huge here:** The average (mean) sale is around **$229.85**, but the median sale is only **$54.49**. Why the big gap? The data is super right-skewed—most orders are small everyday purchases, but a few massive orders pull the average way up. Median gives a much better picture of a typical order.
* **Outliers are high-value B2B orders:** Using the IQR method ($Q3 + 1.5 \times \text{IQR}$), any order over **$498.93** gets flagged as a statistical outlier. That turned out to be **1,167 orders (~11.7% of total volume)**. Instead of deleting them, I kept them because these represent major corporate client orders that bring in serious revenue.
* **High discounts ruin profits:** Looking at the scatter plot between Discount and Profit, there's a clear tipping point. Whenever discounts go past **20%**, profit almost always drops below $0. Basically, aggressive promotions are costing the store money.
* **Best Category:** **Technology** brings in the highest individual order values and profit, while **Office Supplies** sells the highest number of total items.

---

##  PostgreSQL Business Analytics

After cleaning the data in Python, I pushed the DataFrame directly into PostgreSQL using SQLAlchemy to write advanced business queries.

### 1. Revenue & Performance Breakdown by Category & Sub-Category
I used aggregate functions (`COUNT`, `SUM`, `AVG`) and PostgreSQL type casting (`::numeric`) to aggregate overall order counts, revenue, average order value, and profit across product categories:

```sql
SELECT 
    "Category",
    "Sub-Category",
    COUNT("Order ID") AS total_orders,
    ROUND(SUM("Sales")::numeric, 2) AS total_revenue,
    ROUND(AVG("Sales")::numeric, 2) AS avg_order_value,
    ROUND(SUM("Profit")::numeric, 2) AS total_profit
FROM orders 
GROUP BY "Category", "Sub-Category"
ORDER BY total_revenue DESC;
```

### 2. Customer Segmentation
To break down who our best customers are, I wrote a query using a Common Table Expression (CTE) and a `CASE WHEN` statement to bucket people into spending tiers based on their total spend:

```sql
WITH customer_summary AS (
    SELECT
        "Customer ID",
        "Customer Name",
        COUNT("Order ID") AS total_orders,
        ROUND(SUM("Sales")::numeric, 2) AS total_revenue,
        ROUND(AVG("Sales")::numeric, 2) AS avg_order_value
    FROM orders
    GROUP BY "Customer ID", "Customer Name"
)
SELECT 
    "Customer ID",
    "Customer Name",
    total_orders,
    total_revenue,
    CASE 
        WHEN total_revenue >= 3000 THEN 'VIP Tier'
        WHEN total_revenue BETWEEN 1000 AND 2999 THEN 'Regular Tier'
        ELSE 'Low Spender'
    END AS customer_segment
FROM customer_summary
ORDER BY total_revenue DESC;
```

Isolating VIPs (spending $3,000+) allows marketing to build exclusive loyalty incentives while giving lower tiers targeted nudges to re-engage.

### 3. Customer Purchase Sequence 

To track customer retention and chronological order patterns, I used the DENSE_RANK() window function partitioned by customer:

```sql
select 
	"Customer ID",
	"Order ID",
	"Order Date",
	"Sales",
	dense_rank() over (
		partition by "Customer ID"
		order by "Order Date" ASC
	) AS customer_order_sequence
from orders ;	
```

Ranking order sequences allows us to identify a customer's first purchase versus repeat visits, making it easy to analyze purchase frequency and retention over time.
