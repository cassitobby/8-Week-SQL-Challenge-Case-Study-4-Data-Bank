<div align="center">
  <h1><b>Case Study #4: Data Bank</b></h1>
</div>

## D. Extra Challenge

Data Bank wants to try another option which is a bit more difficult to implement - they want to calculate data growth using an interest calculation, just like in a traditional savings account you might have with a bank.

If the annual interest rate is set at 6% and the Data Bank team wants to reward its customers by increasing their data allocation based off the interest calculated on a daily basis at the end of each day, how much data would be required for this option on a monthly basis?

Special notes:

Data Bank wants an initial calculation which does not allow for compounding interest, however they may also be interested in a daily compounding interest calculation so you can try to perform this calculation if you have the stamina!


 ```
SELECT * FROM customer_transactions order by customer_id, txn_date

WITH DayCalCte AS(
	SELECT
		customer_id,
		txn_date,
		txn_amount,
		DATE_TRUNC('month', txn_date) AS month_start_date,
		 EXTRACT(DAY FROM (DATE_TRUNC('MONTH', txn_date) + INTERVAL '1 MONTH' - INTERVAL '1 DAY')) AS days_in_month,
		 EXTRACT(DAY FROM (DATE_TRUNC('MONTH', txn_date) + INTERVAL '1 MONTH' - INTERVAL '1 DAY')) - EXTRACT(DAY FROM txn_date) AS Day_interval
	FROM
		customer_transactions
	WHERE txn_type = 'deposit'
	GROUP BY
		customer_id,
		txn_date,
		txn_amount
	ORDER BY
		customer_id
),
InterestCte AS(
	SELECT 
		customer_id,
		txn_date,
		--txn_amount,
		ROUND(txn_amount * (0.06 / 365) * Day_interval, 2) AS daily_interest_data
	FROM DayCalCte
	GROUP BY
		customer_id,
		txn_date,
		--txn_amount,
		Day_interval,
		txn_amount
	ORDER BY customer_id
)
SELECT 
	EXTRACT(MONTH FROM txn_date) AS month,
	SUM(daily_interest_data) AS monthly_data_required
FROM InterestCte
GROUP BY month
ORDER BY month;
```


**Output**

![image](https://github.com/cassitobby/SQL-challenge-Case-Study--4-Data-Bank/assets/128924056/459447ae-13ea-4e32-a7de-1d33e45c4df1)

**Insight**
- January: Data allocation of 973.86
- February: Data allocation of 845.68
- March: Data allocation of 982.47
- April: Data allocation of 560.17

1. There is a decrease in data allocation from January (973.86) to February (845.68).
2. There is an increase in data allocation from February (845.68) to March (982.47).
3. There is a decrease in data allocation from March (982.47) to April (560.17).
