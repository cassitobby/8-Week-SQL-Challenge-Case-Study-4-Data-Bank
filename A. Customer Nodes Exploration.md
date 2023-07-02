 <div align="center">
  <h1><b>Case Study #4: Data Bank</b></h1>
</div>

## A. Customer Nodes Exploration

## Case Study Questions
1. How many unique nodes are there on the Data Bank system?
2. What is the number of nodes per region?
3. How many customers are allocated to each region?
4. How many days on average are customers reallocated to a different node?
5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
   ***

**1. How many unique nodes are there on the Data Bank system?**

- Use COUNT DISTINCT to count the number of unique nodes

```
SELECT
	COUNT(DISTINCT node_id) AS unique_node
FROM customer_nodes;
```

### Output
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/f4c0786a-a5c3-46c3-80ae-4e111f8164c0)

There are 5 unique noes on Data Bank System
***

**2. What is the number of nodes per region?**
- Use COUNT to count the number of nodes per Region then INNER JOIN to join the Customer_nodes Table to the Regions Table

- Use COUNT DISTINCT to count the number of unique nodes

```
SELECT 
	c.region_id, 
	region_name,
	COUNT(c.node_id) AS num_of_nodes
FROM customer_nodes AS c
INNER JOIN regions AS r
USING (region_id)
GROUP BY 
	c.region_id, 
	region_name
ORDER BY num_of_nodes DESC;
```

### Output
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/f43d42f2-bf6a-4ca1-8ad8-00d3a3242e98)

Australia has the highest number of node(770). Europe has the least number of nodes(616)

***
**3. How many customers are allocated to each region?**

- Use COUNT DISTINCT to count the number of customers allocated to each region, then INNER JOIN to join the Customer_nodes Table to the Regions Table

```
SELECT 
	region_id, 
	region_name,
	COUNT(DISTINCT customer_id) AS num_of_customer
FROM customer_nodes
INNER JOIN regions AS r
USING (region_id)
GROUP BY region_id, region_name
ORDER BY region_id;
```

### Output
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/1df8e200-a2de-4a70-bd91-040c0adc11b1)

Among the regions, Australia has the largest customer allocation count of 110, followed by America with 105, while Europe has the lowest count of 88
***

**4. How many days on average are customers reallocated to a different node?**

- Use the AVG function on the difference between the end_date and start_date.
-  Notice an abnormal date '9999-12-31'. Filter the result to exclude this date
```
SELECT 
	AVG(end_date - start_date) AS avg_reallocation_days
FROM customer_nodes
WHERE end_date != '9999-12-31';

```
### Output
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/dea9e2e2-6814-451c-8afb-1051eb43234e)


It takes 14 days on a average for customers to be reallocated to a different node
***
**5. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**

- Use CTE to calculate reallocation days
- Use PERCENTILE_CONT and WITHIN GROUP to find the median, 80th,and 95th percentile
```
WITH dayscte AS (
    SELECT 
        c.region_id,
        r.region_name,
        AGE(end_date, start_date) AS reallocation_days
    FROM customer_nodes AS c
    INNER JOIN regions AS r
    USING (region_id)
    WHERE end_date != '9999-12-31'
)
SELECT 
    DISTINCT region_name,
    (SELECT PERCENTILE_CONT(0.5) WITHIN GROUP(ORDER BY reallocation_days) FROM dayscte AS d WHERE d.region_name = dayscte.region_name) AS median,
    (SELECT PERCENTILE_CONT(0.80) WITHIN GROUP(ORDER BY reallocation_days) FROM dayscte AS d WHERE d.region_name = dayscte.region_name) AS percentile_80,
    (SELECT PERCENTILE_CONT(0.95) WITHIN GROUP(ORDER BY reallocation_days) FROM dayscte AS d WHERE d.region_name = dayscte.region_name) AS percentile_95
FROM dayscte;
```
### Output
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/a664b0cd-f7a4-4d2f-aa9c-9513c7242b15)

The median allocation days for all the region is 15, the 95 percentile is 28 while the 80 percentile is 24 Days for Africa and Europe and 23 Days for America, Asia and Australia
