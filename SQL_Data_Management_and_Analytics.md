# SQL Scripts for Data Management

## Table Creation

```sql
CREATE TABLE membership(
	membership_id SERIAL PRIMARY KEY,
	start_date DATE,
	end_date DATE
);

SELECT * FROM membership;

CREATE TABLE customers(
	customer_id SERIAL PRIMARY KEY,
	membership_id INTEGER,
	first_name CHARACTER VARYING (200),
	last_name CHARACTER VARYING (200),
	email CHARACTER VARYING (300),
	type CHARACTER VARYING (200),
	address CHARACTER VARYING (300),
	contact_number CHARACTER VARYING (20),
	FOREIGN KEY (membership_id) REFERENCES membership (membership_id) ON UPDATE CASCADE ON DELETE CASCADE
);

SELECT * FROM customers;

CREATE TABLE employees(
	employee_id SERIAL PRIMARY KEY,
	first_name CHARACTER VARYING (200),
	last_name CHARACTER VARYING (200),
	position CHARACTER VARYING (200),
	address CHARACTER VARYING (300),
	branch CHARACTER VARYING (10),
	contact_number CHARACTER VARYING (20)
);

SELECT * FROM employees;

CREATE TABLE shipment_details(
	shipment_id SERIAL PRIMARY KEY,
	customer_id INTEGER,
	shipment_content CHARACTER VARYING (200),
	shipment_domain CHARACTER VARYING (100),
	service_type CHARACTER VARYING (100),
	shipment_weight NUMERIC,
	shipment_charges NUMERIC,
	origin_addr CHARACTER VARYING (300),
	destination_addr CHARACTER VARYING (300),
	FOREIGN KEY (customer_id) REFERENCES customers (customer_id) ON UPDATE CASCADE ON DELETE CASCADE
);

SELECT * FROM shipment_details;

CREATE TABLE employees_manage_shipments(
	employee_shipment_id SERIAL PRIMARY KEY, 
	employee_id INTEGER,
	shipment_id INTEGER,
	FOREIGN KEY (shipment_id) REFERENCES shipment_details (shipment_id) ON UPDATE CASCADE ON DELETE CASCADE,
	FOREIGN KEY (employee_id) REFERENCES employees (employee_id) ON UPDATE CASCADE ON DELETE CASCADE
);

SELECT * FROM employees_manage_shipments;

CREATE TABLE status(
	status_id SERIAL PRIMARY KEY,
	shipment_id INTEGER,
	current_status CHARACTER VARYING (100),
	shipping_date DATE,
	delivery_date DATE,
	FOREIGN KEY (shipment_id) REFERENCES shipment_details (shipment_id) ON UPDATE CASCADE ON DELETE CASCADE
);

SELECT * FROM status;

CREATE TABLE payment(
	payment_id CHARACTER VARYING (100) PRIMARY KEY,
	customer_id INTEGER,
	shipment_id INTEGER,
	amount NUMERIC,
	payment_status CHARACTER VARYING (100),
	payment_mode CHARACTER VARYING (100),
	payment_date DATE,
	FOREIGN KEY (customer_id) REFERENCES customers (customer_id) ON UPDATE CASCADE ON DELETE CASCADE,
	FOREIGN KEY (shipment_id) REFERENCES shipment_details (shipment_id) ON UPDATE CASCADE ON DELETE CASCADE
);

SELECT * FROM payment;
```

## Queries

### Query 1: Total Shipments and Average Weight by Shipment Content
```sql
WITH content_shipments AS(
	SELECT
		shipment_content,
		SUM(shipment_weight) AS total_shipment_weight,
		COUNT(shipment_id) AS total_shipments
	FROM shipment_details 
	GROUP BY shipment_content
)
SELECT 
	shipment_content,
	total_shipments,
	ROUND(total_shipment_weight / total_shipments, 2) AS avg_weight_per_shipment
FROM content_shipments
ORDER BY avg_weight_per_shipment DESC;
```

### Query 2: Membership Trend by Ending Years (from 2024)
```sql
WITH active_members AS (
	SELECT
		membership_id,
		start_date,
		end_date
	FROM membership
	WHERE end_date > DATE(now())
)
SELECT
	COUNT(membership_id) AS membership_ending_count,
	EXTRACT(YEAR FROM end_date) AS membership_end_year,
	EXTRACT(MONTH FROM end_date) AS membership_end_month
FROM active_members
GROUP BY EXTRACT(YEAR FROM end_date), EXTRACT(MONTH FROM end_date)
ORDER BY EXTRACT(YEAR FROM end_date);
```

### Query 3: Revenue by Membership Expiration Year
```sql
WITH active_members AS (
	SELECT
		membership_id,
		start_date,
		end_date
	FROM membership
	WHERE end_date > DATE(now())
)
SELECT
	act_mem.membership_id,
	pay.amount,
	pay.payment_status,
	SUM(pay.amount) OVER(PARTITION BY EXTRACT(YEAR FROM act_mem.end_date)) AS total_amount_by_year,
	ROUND(amount / SUM(pay.amount) OVER(PARTITION BY EXTRACT(YEAR FROM act_mem.end_date)), 2) AS ratio_amount_by_year,
	SUM(pay.amount) OVER(PARTITION BY pay.payment_status) AS total_amount_by_status,
	ROUND(amount / SUM(pay.amount) OVER(PARTITION BY pay.payment_status), 2) AS ratio_amount_by_status,
	EXTRACT(YEAR FROM act_mem.end_date) AS membership_end_year
FROM active_members act_mem
LEFT JOIN customers cus ON act_mem.membership_id = cus.membership_id
LEFT JOIN payment pay ON cus.customer_id = pay.customer_id
ORDER BY EXTRACT(YEAR FROM act_mem.end_date), pay.payment_status;
```

### Query 4: Shipment Charges Analysis
```sql
SELECT
	shipment_content,
	shipment_domain,
	service_type,
	SUM(shipment_charges) AS charges
FROM shipment_details
GROUP BY CUBE(shipment_content, shipment_domain, service_type);
```

### Query 5: Categorize Customers by Membership Duration
```sql
WITH CTE_CUST_MEM AS (
	SELECT 
		CONCAT_WS(' ', cus.first_name, cus.last_name) AS customer, 
		mem.start_date, 
		mem.end_date, 
		ROUND((DATE(now()) - mem.start_date) / 365.0, 2) AS membership_duration_years
	FROM customers cus
	INNER JOIN membership mem ON mem.membership_id = cus.membership_id
	WHERE EXTRACT(YEAR FROM mem.end_date) >= 2024 AND EXTRACT(MONTH FROM mem.end_date) >= 8
)
SELECT 
	customer, 
	membership_duration_years,
	CASE 
		WHEN membership_duration_years < 5 THEN 'BRONZE'
		WHEN membership_duration_years BETWEEN 5 AND 9.99 THEN 'SILVER'
		WHEN membership_duration_years BETWEEN 10 AND 14.99 THEN 'GOLD'
		WHEN membership_duration_years > 15 THEN 'PLATINUM'
	END AS category
FROM CTE_CUST_MEM
ORDER BY membership_duration_years DESC;
```

### Query 6: Top 15 Branches by Delivery Ratio
```sql
WITH CTE_EMP_SHIP_STAT AS (
	SELECT 
		e.branch,
		CASE s.current_status
			WHEN 'Delivered' THEN 1
			WHEN 'Not Delivered' THEN 0
		END AS current_status
	FROM employees e
	INNER JOIN employees_manage_shipments ems ON ems.employee_id = e.employee_id
	INNER JOIN shipment_details sh ON sh.shipment_id = ems.shipment_id
	INNER JOIN status s ON s.shipment_id = sh.shipment_id
	WHERE EXTRACT(YEAR FROM s.shipping_date) < 2024
)
SELECT 
	branch, 
	ROUND((CAST(delivered AS DECIMAL) / total) * 100, 2) AS delivery_ratio_perc
FROM (
	SELECT 
		branch, 
		SUM(current_status) AS delivered, 
		COUNT(branch) AS total
	FROM CTE_EMP_SHIP_STAT
	GROUP BY branch
) AS branch_stats
ORDER BY delivery_ratio_perc DESC
LIMIT 15;
```

### Query 7: Pending Debt by Shipping Domain and Customer
```sql
SELECT 
	sh.shipment_domain, 
	cus.type AS customer_type, 
	CONCAT_WS(' ', cus.first_name, cus.last_name) AS customer,
	SUM(pay.amount) AS pending_debt
FROM customers cus
INNER JOIN payment pay ON cus.customer_id = pay.customer_id
INNER JOIN shipment_details sh ON sh.shipment_id = pay.shipment_id
INNER JOIN status s ON s.shipment_id = sh.shipment_id
WHERE pay.payment_status = 'NOT PAID'
GROUP BY ROLLUP(sh.shipment_domain, cus.type, customer);
```

### Query 8: Shipments with the Highest Profit Margin
```sql
WITH shipment_costs AS (
	SELECT shipment_id,
		shipment_charges,
		shipment_weight * 2 AS estimated_cost -- Simplified cost model
	FROM shipment_details
)
SELECT sd.shipment_id, sd.customer_id, sd.shipment_content, sd.service_type,
	sc.shipment_charges, sc.estimated_cost, 
	sc.shipment_charges - sc.estimated_cost AS profit, -- Calculated profit
	ROUND((sc.shipment_charges - sc.estimated_cost) / sc.shipment_charges * 100, 2) AS profit_margin, -- Calculated profit margin
	CASE
		WHEN (sc.shipment_charges - sc.estimated_cost) / sc.shipment_charges > 0.50 THEN 'High'
		WHEN (sc.shipment_charges - sc.estimated_cost) / sc.shipment_charges BETWEEN 0.20 AND 0.50 THEN 'Medium'
		ELSE 'Low'
	END AS profit_category -- Categorized profit margin
FROM shipment_details sd
JOIN shipment_costs sc ON sd.shipment_id = sc.shipment_id
ORDER BY profit_margin DESC
LIMIT 100;
```

### Query 9: Yearly Revenue Trends with Running Total
```sql
WITH revenue_per_year AS (
	SELECT EXTRACT(YEAR FROM payment_date) AS year,
		SUM(amount) AS yearly_revenue
	FROM payment
	GROUP BY EXTRACT(YEAR FROM payment_date)
)
SELECT year,
	yearly_revenue,
	SUM(yearly_revenue) OVER (ORDER BY year) AS running_total_revenue
FROM revenue_per_year
ORDER BY year;
```

### Query 10: Ranking Customers by Total Shipment Charges
```sql
SELECT customer_id, CONCAT_WS(' ', first_name, last_name) AS full_name, total_charges,
	RANK() OVER (ORDER BY total_charges DESC) AS charges_ranks
FROM (
	SELECT c.customer_id, c.first_name, c.last_name, 
		SUM(s.shipment_charges) AS total_charges
	FROM customers c
	JOIN shipment_details s ON c.customer_id = s.customer_id
	GROUP BY c.customer_id, c.first_name, c.last_name
) AS customer_charges
ORDER BY charges_ranks ASC
LIMIT 20;
```
