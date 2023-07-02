<div align="center">
  <h1><b>Case Study #4: Data Bank</b></h1>
</div>

## B. Customer Transactions

## Case Study Questions
1. What is the unique count and total amount for each transaction type?
2. What is the average total historical deposit counts and amounts for all customers?
3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
4. What is the closing balance for each customer at the end of the month?
5. What is the percentage of customers who increase their closing balance by more than 5%?
***
**1. What is the unique count and total amount for each transaction type?**
   - Use count to get the count of transaction type and SUM to add the transaction amount for each type
   
 ```
SELECT 
	txn_type, 
	COUNT(*) AS num_of_transactions, 
	SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type
ORDER BY num_of_transactions DESC;

```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/ef63ca55-b2f0-4522-8324-73e86e01d003)

The transaction type "Deposit" has the highest count, with a total of 2,771 occurrences and an accumulated amount of 1,359,168. Following closely is the transaction type "Purchase," which has been recorded 1,617 times, with a total amount of 806,537. On the other hand, "Withdrawal" has the lowest count of 1,580 instances, with a total amount of 793,003.

***

**2. What is the average total historical deposit counts and amounts for all customers?**
 - create a CTE called summaryCte that computes the total sum and count of transactions with txn_type "deposit".
 - Formulate a query to retrieve the average and count of transactions for txn_type from the summaryCte.
   
 ```
WITH summaryCte AS (
	SELECT 
		customer_id,
		txn_type,
		COUNT(CASE
				WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS deposit_count,
		SUM(CASE
				WHEN txn_type = 'deposit' THEN txn_amount END) AS deposit_amount
	FROM customer_transactions
	GROUP BY
		customer_id, txn_type
)
SELECT 
	txn_type,
	AVG(deposit_count) AS avg_deposit_count,
	AVG(deposit_amount) AS avg_deposit_amount
FROM summaryCte
WHERE txn_type = 'deposit'
GROUP BY txn_type;
```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/9f0d6612-fa68-461a-bdac-2572ff511abd)

The average historical deposit count is 5.34 while the average amount is	2718.33

***
**3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**
   - Create a CTE called CountCte that counts the number of times each transaction type(txn_type) occur.
   - Write another query that filters the CountCte to include only those records where the count for "deposit" is greater than 1, and either the count for "purchase" or the count for "withdrawal" is greater than 0.
   - Group the result by month
   
 ```
WITH CountCte AS(
	SELECT 
		TO_CHAR(txn_date, 'Mon') AS month,
		customer_id,
		COUNT(CASE
				WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS deposit_count,
		COUNT(CASE
			 WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS purchase_count,
		COUNT(CASE
			 WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawal_count
	FROM customer_transactions
	GROUP BY customer_id, month
)
SELECT 
	month, 
	COUNT(DISTINCT customer_id) AS customer_count
FROM CountCte
WHERE deposit_count > 1
AND (purchase_count >0 OR withdrawal_count > 0)
GROUP BY month
ORDER BY customer_count DESC;
```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/d2e5ce16-8ac3-48ff-bd9d-0753847af6bb)

The customer count was highest in March with 393 customers, followed by February with 384 customers, and January with 353 customers. However, there was a noticeable decrease in April, where the customer count dropped significantly to 192. 

***

**4. What is the closing balance for each customer at the end of the month?**
   - The transaction types (txn_type) can be categorized into two groups: inflow and outflow. "Deposit" falls under the inflow category as it contributes to an increase in our balance. Conversely, "Purchase" and "Withdrawal" belong to the outflow category. These transactions have a negative impact on our balance, resulting in a reduction.
   - Creates a CTE named AmountCte that calculates the amount for each customer and month. It considers different transaction types and calculates the sum of the amount based on whether it is a deposit (positive) or purchase/withdrawal (negative).
   - The main query selects the customer ID, month, and calculates the closing balance using the cumulative sum of the amount for each customer. The closing_balance is calculated by summing the amount from the beginning until the current row.
   
 ```
WITH AmountCte AS(
	SELECT 
		customer_id,
		EXTRACT(MONTH FROM txn_date) AS month,
		SUM(CASE 
		WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS amount
	FROM customer_transactions
	GROUP BY customer_id, month
	ORDER BY customer_id
)
SELECT 
	customer_id, 
	month,
	SUM(amount)OVER(PARTITION BY customer_id ORDER BY MONTH ROWS BETWEEN
				   UNBOUNDED PRECEDING AND CURRENT ROW) AS closing_balance
FROM AmountCte
GROUP BY customer_id, month, amount
ORDER BY customer_id;
```
**Output**

**Please note** that due to the potentially large number of results, not all output is displayed to avoid occupying excessive space.

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/1a64d44b-bb91-49e0-9899-09175f26afb5)

***


**5. What is the percentage of customers who increase their closing balance by more than 5%?**
   - 
   
 ```
WITH AmountCte AS(
	SELECT 
		customer_id,
		EXTRACT(MONTH FROM txn_date) AS month,
		SUM(CASE 
		WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS amount
	FROM customer_transactions
	GROUP BY customer_id, month
	ORDER BY customer_id
),
ClosingBalance AS(
	SELECT 
		customer_id, 
		month,
		SUM(amount)OVER(PARTITION BY customer_id, month ORDER BY MONTH ROWS BETWEEN
					   UNBOUNDED PRECEDING AND CURRENT ROW) AS closing_balance
	FROM AmountCte
	GROUP BY customer_id, month, amount
	ORDER BY customer_id
),
PercentageIncrease AS (
	SELECT 
		customer_id,
		month,
		closing_balance,
		100 *(closing_balance - LAG(closing_balance) OVER(PARTITION BY customer_id ORDER BY	month))
			 / NULLIF( LAG(closing_balance) OVER(PARTITION BY customer_id ORDER BY	month), 2) AS percentage_increase
	FROM ClosingBalance
)
SELECT 100 * COUNT(DISTINCT customer_id)/ (SELECT COUNT(DISTINCT customer_id) FROM customer_transactions)::float AS percentage_cutomer
FROM PercentageIncrease
WHERE percentage_increase > 5;
```
**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/067d388a-97e4-4a14-8b45-16e64874d905)

More than half (53.8%) of the customers have a closing balance exceeding 5%.
