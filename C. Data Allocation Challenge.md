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

**1. running customer balance column that includes the impact each transaction**
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

**2. customer balance at the end of each month**

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

