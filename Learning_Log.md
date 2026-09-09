# SQL Learning Log

## Entry 1 - Duplicate customer_id trap

**Question:** Count orders per customer, filter for customers with more
than 3 orders, sort by order count descending.

**Query:**
```sql
SELECT COUNT(order_id), customer_unique_id
FROM olist_orders_dataset
JOIN olist_customers_dataset
  ON olist_orders_dataset.customer_id = olist_customers_dataset.customer_id
GROUP BY customer_unique_id
HAVING COUNT(order_id) > 3
ORDER BY COUNT(order_id) DESC;
```

**Bug:** Grouped by customer_id, got 0 rows with HAVING > 3.
**Root cause:** customer_id in the orders table is per-order, not per-person.
The real customer identity is customer_unique_id, in a separate customers table.
**Fix:** JOIN orders to customers, GROUP BY customer_unique_id instead.
**Takeaway:** Never trust an ID column's name blindly — verify cardinality
(run a quick COUNT/GROUP BY check) before building logic on top of it.

## Entry 2 - Average order value, aggregate without GROUP BY

**Question:** For each customer, find their average order value, using
customer_unique_id, ranked highest average first.

**Query:**
```sql
SELECT AVG(total_payment) AS avg_order_value, olist_customers_dataset.customer_unique_id
FROM (SELECT order_id, SUM(payment_value) AS total_payment
      FROM olist_order_payments_dataset
      GROUP BY order_id) AS x
JOIN olist_orders_dataset ON x.order_id = olist_orders_dataset.order_id
JOIN olist_customers_dataset ON olist_customers_dataset.customer_id = olist_orders_dataset.customer_id
GROUP BY customer_unique_id
ORDER BY avg_order_value DESC;
```

**Bug 1:** Directly averaged raw payment_value rows — this double-counts
orders with multiple installments, skewing the average.
**Fix 1:** SUM(payment_value) grouped by order_id first, collapsing
installments into one total per order — via a subquery.
**Bug 2:** Ran AVG on the joined result with no GROUP BY — got 1 row,
one giant average across all customers instead of per-customer.
**Fix 2:** Added GROUP BY customer_unique_id to the outer query.
**Takeaway:** Aggregate functions (AVG, COUNT, SUM) collapse ALL rows into
one result unless GROUP BY tells SQL where to draw the group boundaries.
Also: when data has a one-to-many relationship (order → payments),
collapse the "many" side first before joining outward.

**Entry 3 - Ranking orders per customer (window functions)**

**Question:** For each customer, rank their own orders by value, highest to lowest. Ranking restarts for every new customer.

**Query:**
```sql
SELECT customer_unique_id, olist_orders_dataset.order_id, total_payment,
RANK() OVER (PARTITION BY customer_unique_id ORDER BY total_payment DESC) AS order_rank
FROM (SELECT order_id, SUM(payment_value) AS total_payment
      FROM olist_order_payments_dataset
      GROUP BY order_id) AS x
JOIN olist_orders_dataset ON x.order_id = olist_orders_dataset.order_id
JOIN olist_customers_dataset ON olist_orders_dataset.customer_id = olist_customers_dataset.customer_id;
```

**Takeaway:** GROUP BY merges rows into one. Window functions (RANK, OVER, PARTITION BY) keep every row, just add a number to each one. PARTITION BY means "restart the count every time this column changes" — here, every new customer.

---

**Entry 4 - Missing payment record breaks the count**

**What happened:** My average-order-value query returned 96095 customers one day, 96096 the next. Same query, same logic.

**Why:** One order in the whole dataset has zero rows in the payments table. My JOIN was a normal JOIN, so it silently dropped that one order — and the customer who only had that one order disappeared from the results too.

**How I found it:**
```sql
SELECT o.order_id, o.customer_id
FROM olist_orders_dataset o
LEFT JOIN olist_order_payments_dataset p ON o.order_id = p.order_id
WHERE p.order_id IS NULL;
```
LEFT JOIN keeps all orders even with no match, so `WHERE ... IS NULL` shows exactly which ones have no payment.

**Takeaway:** Real data has gaps. If two queries with the same logic give different row counts, don't ignore it — use LEFT JOIN + IS NULL to find what's missing.

---

**Entry 5 - Same ranking query, written with CTE + a GROUP BY fix**

**Question:** Same as Entry 3, but written using `WITH` instead of a nested subquery.

**Query:**
```sql
WITH x AS (
  SELECT customer_unique_id, olist_order_payments_dataset.order_id, SUM(payment_value) AS total_pay
  FROM olist_order_payments_dataset
  JOIN olist_orders_dataset ON olist_order_payments_dataset.order_id = olist_orders_dataset.order_id
  JOIN olist_customers_dataset ON olist_orders_dataset.customer_id = olist_customers_dataset.customer_id
  GROUP BY olist_order_payments_dataset.order_id, customer_unique_id
)
SELECT customer_unique_id, order_id, total_pay,
RANK() OVER (PARTITION BY customer_unique_id ORDER BY total_pay DESC) AS order_rank
FROM x;
```

**Two things learned:**
1. `WITH x AS (...)` works the same as `(...) AS x` in FROM. Just cleaner to read.
2. First version only had `order_id` in GROUP BY, not `customer_unique_id`. SQLite allowed it, but stricter databases (Postgres, MySQL) would reject it. Rule: put every non-summed column from SELECT into GROUP BY too, always — even if SQLite lets you skip it.

## Entry 6 - Delivery delay using date functions

**Question:** For every delivered order, calculate how many days late (or
early) it arrived — compare order_delivered_customer_date against
order_estimated_delivery_date. Positive number = late, negative = early.

**Query:**
```sql
SELECT order_id,
ROUND(julianday(order_delivered_customer_date) - julianday(order_estimated_delivery_date)) AS day_diff
FROM olist_orders_dataset
WHERE order_status = 'delivered';
```

**Bug:** First version had no WHERE filter — ran the date math on every
order, including cancelled/undelivered ones with no delivery date,
producing garbage or NULL results.
**Fix:** Filtered to order_status = 'delivered' before doing any date math.
**Takeaway:** julianday() converts a date into a number so subtraction works.
Always filter to the relevant status/condition before running date math —
irrelevant rows (NULLs, cancelled orders) will corrupt the result silently.

## Entry 7 - MIN/MAX solved what I thought needed a self join

**Question:** Retrieve first order date, latest order date, and the
difference between them for each customer — helps identify which
customers stay dormant.

**Query:**
```sql
SELECT customer_unique_id, first_p, latest_p,
ROUND(julianday(latest_p) - julianday(first_p)) AS time_diff
FROM (
  SELECT customer_unique_id,
  MIN(order_purchase_timestamp) AS first_p,
  MAX(order_purchase_timestamp) AS latest_p
  FROM olist_customers_dataset
  JOIN olist_orders_dataset ON olist_customers_dataset.customer_id = olist_orders_dataset.customer_id
  GROUP BY customer_unique_id
)
ORDER BY time_diff DESC;
```

**Bug:** Tried to use an alias (first_p) in the same SELECT line where it
was defined, without a subquery. SQL can't reference an alias before it's
resolved in that scope.
**Fix:** Wrapped the MIN/MAX query as a subquery, then used the aliases
in the outer SELECT where they now exist.
**Data insight:** Most date differences came out as 0 — meaning most
customers placed exactly one order and never returned. A few customers
showed gaps of 60, 177, 330+ days.
**Takeaway:** Before reaching for an advanced technique (self join, CTE),
check if a simpler tool already known (GROUP BY + aggregate functions)
solves the problem first.
