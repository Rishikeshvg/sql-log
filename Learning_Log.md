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
