<div align="center">
  <h1><b>Case Study #4: Data Bank</b></h1>
</div>

## C. Data Allocation Challenge

## Case Study Questions
To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 3 different options:

- Option 1: data is allocated based off the amount of money at the end of the previous month
- Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days
- Option 3: data is updated real-time
  
For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:

- running customer balance column that includes the impact each transaction
- customer balance at the end of each month
- minimum, average and maximum values of the running balance for each customer
Using all of the data available - how much data would have been required for each option on a monthly basis?

***

**- running customer balance column that includes the impact each transaction**
   - First, create a CTE named AmountCte that reflects the impact of the transaction type
   - Use  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW to bring forward the balance of the previous month. 
   
 ```
WITH AmountCte AS (
	SELECT
		customer_id,
		txn_date,
		CASE 
			WHEN txn_type = 'deposit' THEN txn_amount 
			ELSE -txn_amount END AS balance
	FROM customer_transactions
	ORDER BY customer_id
)
SELECT
		customer_id,
		txn_date,
		SUM(balance) OVER(PARTITION BY customer_id ORDER BY txn_date 
						  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM AmountCte
GROUP BY customer_id, txn_date, balance
ORDER BY customer_id;
```
**Output**

**Please note** that due to the potentially large number of results, not all output is displayed to avoid occupying excessive space.
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/38a7f267-dc94-43b0-b009-1f892e3a9fd3)

***

**- customer balance at the end of each month**

 ```
ITH AmountCte AS(
	SELECT 
		customer_id,
		EXTRACT(MONTH from txn_date) AS month,
		SUM(CASE
			WHEN txn_type = 'deposit' THEN txn_amount
			ELSE -txn_amount END) as amount
	FROM customer_transactions
	GROUP BY customer_id, month
	ORDER BY customer_id, month
)
SELECT 
	customer_id,
	month,
	SUM(amount) OVER(PARTITION  BY customer_id, month ORDER BY month 
						 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
FROM AmountCte
GROUP BY customer_id, month, amount
```
**Output**

**Please note** that due to the potentially large number of results, not all output is displayed to avoid occupying excessive space.
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/2d0cf537-30b9-4410-85a3-d303edfd41b6)

***

**- minimum, average and maximum values of the running balance for each customer**
   - Create a CTE named RunningTotalCte that reflects the effect each transactions and running  balance for each.
   - Then write an outter query that finds the Min, Max and Avg value of each transactions from RunningTotalCte.
   
 ```
WITH RunningTotalCte AS(
SELECT
	customer_id,
	txn_date,
	SUM(CASE 
			WHEN txn_type = 'deposit' THEN txn_amount 
			WHEN txn_type = 'purchase' THEN -txn_amount 
			WHEN txn_type = 'withdrawal' THEN -txn_amount ELSE 0 END)
			OVER(PARTITION BY customer_id ORDER BY txn_date 
				 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM customer_transactions
)
SELECT 
	customer_id, 
	MIN(running_total) AS min_running_total,
	AVG(running_total) AS avg_running_total,
	MAX(running_total) AS max_running_total
FROM RunningTotalCte
GROUP BY customer_id; 
```
**Output**

**Please note** that due to the potentially large number of results, not all output is displayed to avoid occupying excessive space.
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/bbf3f896-52cf-4339-a4f0-844bb104fc19)

***

**- Option 1: data is allocated based off the amount of money at the end of the previous month**
   - Create the first Common Table Expression (CTE) called "AmountCte" to determine the impact of each transaction. We assigned negative values to Withdrawals and Purchases.
   - Next, we employed the second CTE, "RunningBalance," to calculate the running balance. It's worth noting that the balance at the end of a month is carried forward to the beginning of the following month.
   - To address the need for allocation based on the money at the end of the previous month, we introduced a MonthlyAllocation CTE. This CTE was used to lag the running balance, allowing us to access the balance from the preceding month.
   - In order to obtain the total allocation, we utilized an outer query to sum the "total_allocation" values.
   
 ```
WITH AmountCte AS (
	SELECT 
		customer_id,
		EXTRACT(MONTH from txn_date) AS month,
		CASE
			WHEN txn_type = 'deposit' THEN txn_amount
			WHEN txn_type = 'purchase' THEN -txn_amount
			WHEN txn_type = 'withdrawal' THEN -txn_amount END as amount
	FROM customer_transactions
	ORDER BY customer_id, month
),
RunningBalance AS (
	SELECT 
		*,
		SUM(amount) OVER (PARTITION BY customer_id, month ORDER BY customer_id, month
			   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
	FROM AmountCte
),
MonthlyAllocation AS(
	SELECT 
		*,
		LAG(running_balance, 1) OVER(PARTITION BY customer_id 
									 ORDER BY customer_id, month) AS monthly_allocation
	FROM RunningBalance
)
SELECT
	month,
	SUM(monthly_allocation) AS total_allocation
FROM MonthlyAllocation
GROUP BY month
ORDER BY month;
```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/56194a64-9ab3-4b57-8532-c6daa046fb61)

Upon observation, it was noted that the data allocation based on the previous month's balance resulted in negative values for each month. This occurred due to customers who had a negative balance in the previous month. However, it is not feasible for these customers to receive a negative data storage allocation for subsequent months. Therefore, we made the **assumption that customers with a negative previous month balance would receive a data storage allocation of 0 for the next month.**


Considering the assumption, the outer query was modified to allocate 0 data storage for customers that have a balance less than 0 for previous month

 ```
SELECT
	month,
	SUM(
		CASE WHEN monthly_allocation < 0 THEN 0 ELSE monthly_allocation END) AS total_allocation
FROM MonthlyAllocation
GROUP BY month
ORDER BY month;
```
![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/b407f021-a74d-4bad-af31-9a615c5b7d95)

**Insight**
- The data allocation for February (542,109) is significantly higher than that of January (350,395). This indicates an increase in data storage needs or a higher demand for data resources in February.
- March (490,210) shows a slightly lower data allocation compared to February (542,109), suggesting a potential stabilization or a slight decrease in data storage requirements during that month.
- April (252,704) exhibits a considerable drop in data allocation compared to the previous months.
- The overall trend suggests a fluctuation in data allocations throughout the observed period, indicating varying levels of data usage or requirements over time.

***

**- Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days**
   - Use AmountCte CTE to calculate the net amount
   - Use ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW on another CTE called RunningBalance to calculate the running balance for each month
   - Use Avg_running_balance CTE to calculate the average running balance
   -  Use an outter query to sum the averages. 
   
 ```
WITH AmountCte AS (
	SELECT 
		customer_id,
		EXTRACT(MONTH from txn_date) AS month,
		SUM(CASE
			WHEN txn_type = 'deposit' THEN txn_amount
			WHEN txn_type = 'purchase' THEN -txn_amount
			WHEN txn_type = 'withdrawal' THEN -txn_amount END) as net_amount
	FROM customer_transactions
	GROUP BY customer_id, month
),
RunningBalance AS (
	SELECT 
		*,
		SUM(net_amount) OVER (PARTITION BY customer_id ORDER BY month
			   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_balance
	FROM AmountCte
	ORDER BY customer_id, month
),
Avg_running_balance AS(
	SELECT customer_id, 
	month,
	AVG(running_balance) OVER(PARTITION BY customer_id) AS avg_balance
	FROM RunningBalance
	GROUP BY customer_id, month, running_balance
	ORDER BY customer_id
)
SELECT 
	month,
	ROUND(SUM(CASE
			WHEN avg_balance < 0 THEN 0 ELSE avg_balance END), 2) AS data_needed_per_month
FROM Avg_running_balance
GROUP BY month
ORDER BY month	
```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/9b8d7826-ee75-4594-810b-5981e5c39274)

**Insights**

- The average allocation for January is 218,196.92. This represents the average amount allocated per month during that period.
- In February, the average allocation decreases to 196,413.42. This suggests a potential decrease in the average data storage requirements or a change in data management strategies compared to January.
- March shows a slight increase in the average allocation, with a value of 201,383.42. This indicates a potential stabilization or a slight increase in the average data storage needs during that month.
- April exhibits a significant drop in the average allocation, with a value of 124,976.25. This suggests a considerable decrease in the average data storage requirements or a notable shift in data management practices compared to the previous months.

***

**- Option 3: data is updated real-time**
   - 
   - 
   
 ```
WITH AmountCte AS (
	SELECT
		customer_id,
		EXTRACT(MONTH FROM txn_date) AS month,
		SUM(CASE 
			WHEN txn_type = 'deposit' THEN txn_amount 
			ELSE -txn_amount END) AS balance
	FROM customer_transactions
	GROUP BY customer_id, month
),
Running_balance AS (
	SELECT
			customer_id,
			month,
			SUM(balance) OVER(PARTITION BY customer_id ORDER BY month 
							  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
	FROM AmountCte
	GROUP BY customer_id, month, balance
)
SELECT 
	month, 
	SUM(CASE WHEN running_total < 0 THEN 0 ELSE running_total END) Data_required
FROM Running_balance
GROUP BY month
ORDER BY month;

```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/296c1b45-2c73-455c-8920-8ea91953aeb4)

**Insight**

- Month 3 has a data allocation of 240,065. This represents the data allocation for that specific month based on real-time updates.
- Month 1 has a data allocation of 235,595, which indicates the data allocation for that month based on real-time updates.
- Month 4 shows a decrease in the data allocation compared to the previous months, with a value of 157,033.
- Month 2 has a data allocation of 238,492, representing the data allocation for that specific month based on real-time updates.
