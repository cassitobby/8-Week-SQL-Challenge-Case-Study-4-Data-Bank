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

**-Option 1: data is allocated based off the amount of money at the end of the previous month**
   - For this part, Use the first CTE AmountCte to calculate the effect of each transaction. Assigning negative to Withdrawal and purchase.
   - Second CTE RunningBalance was use to calculate the running balance. **Note:** The balance at the end of a Month is brought forward in the beginning of another month
   - Since the allocation is based of the money at the end of previous Month, I created a MonthlyAllocation named CTE to lag the running balnce. This is to get the previous month balance.
   - I used an outer query to sum total_allocation
   
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

***

