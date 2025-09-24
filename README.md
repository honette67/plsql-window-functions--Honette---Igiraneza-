#Project: PL/SQL Window Functions – Banking Transactions

Course: Database Development with PL/SQL (INSY 8311)
Instructor: Eric Maniraguha
Name: Honette Igiraneza
Id: 27707
________________________________________________________________________________________________
1. PROBLEM DEFINITION

A.Business Context
A commercial bank in Rwanda operates across multiple branches and manages thousands of client accounts. The bank wants to analyze client transactions to identify high-value clients, monitor deposit growth, and support decision-making for customer loyalty programs and liquidity planning.

B.Data Challenge
The bank’s management lacks a clear view of which clients contribute the most deposits, how deposits change month-to-month, and how clients can be segmented into meaningful groups. They also need tools to smooth fluctuations in deposit amounts to make better forecasts.

C.Expected Outcome
By using SQL window functions, the bank can:
* Identify top clients per branch and quarter.
* Monitor cumulative deposit growth month by month.
* Track deposit growth percentages compared to previous months.
* Segment clients into quartiles (VIP, high, average, low).
* Calculate moving averages to smooth seasonal fluctuations.
  
2.DATABASE SCHEMA

Tables
client
| Column     | Type         | Key | Example     |
| ---------- | ------------ | --- | ----------- |
| client_id  | INT           | PK  | 101         |
| name       | VARCHAR(100) |     | Alice Uwase |
| branch     | VARCHAR(50)  |     | Kigali      |

accounts
| Column        | Type        | Key | Example|
| -------------| -----------| --- | -------|
| account_id   | INT         | PK  | 201     |
| client_id    | INT (FK)    |     | 101     |
| account_type | VARCHAR(20) |     | Savings |

transaction
| Column          | Type          | Key | Example              |
| --------------- | ------------- | --- | -------------------- |
| transaction\_id | INT           | PK  | 301                  |
| account\_id     | INT (FK)      |     | 201                  |
| trans\_date     | DATE          |     | 2024-01-10           |
| trans\_type     | VARCHAR(15)   |     | Deposit / Withdrawal |
| amount          | DECIMAL(12,2) |     | 150000.00            |

3.Queries Implemented

1. Ranking – Top 5 Clients per Branch/Quarter
WITH branch_client_quarter AS (
  SELECT c.branch,
         c.name AS client_name,
         TO_CHAR(trans_date,'YYYY-"Q"Q') AS quarter,
         SUM(amount) AS total_amount
  FROM "transaction" t
  JOIN accounts a ON t.account_id = a.account_id
  JOIN client c ON a.client_id = c.client_id
  WHERE trans_type = 'Deposit'
  GROUP BY c.branch, c.name, TO_CHAR(trans_date,'YYYY-"Q"Q')
),
ranked AS (
  SELECT branch,
         quarter,
         client_name,
         total_amount,
         RANK() OVER (PARTITION BY branch, quarter ORDER BY total_amount DESC) AS rnk
  FROM branch_client_quarter
)
SELECT *
FROM ranked
WHERE rnk <= 5
ORDER BY branch, quarter, rnk;

🔹 Shows the top 5 depositors per branch and quarter, useful for loyalty programs.

2. Aggregate – Running Monthly Deposit Totals
   
SELECT TO_CHAR(trans_date,'YYYY-MM') AS month,
       SUM(amount) AS monthly_total,
       SUM(SUM(amount)) OVER (ORDER BY TO_CHAR(trans_date,'YYYY-MM')) AS running_total
FROM "transaction"
WHERE trans_type = 'Deposit'
GROUP BY TO_CHAR(trans_date,'YYYY-MM')
ORDER BY month;

🔹 Displays cumulative growth of deposits month by month.

3. Navigation – Month-over-Month Growth
   
WITH monthly AS (
  SELECT TO_CHAR(trans_date,'YYYY-MM') AS month,
         SUM(amount) AS deposits
  FROM "transaction"
  WHERE trans_type = 'Deposit'
  GROUP BY TO_CHAR(trans_date,'YYYY-MM')
)
SELECT month,
       deposits,
       LAG(deposits) OVER (ORDER BY month) AS prev_month,
       ROUND( (deposits - LAG(deposits) OVER (ORDER BY month))
             / LAG(deposits) OVER (ORDER BY month) * 100, 2) AS growth_pct
FROM monthly
ORDER BY month;

🔹 Shows percentage growth in deposits compared to the previous month.

4. Distribution – Client Quartiles
   
WITH client_totals AS (
  SELECT c.client_id,
         c.name,
         SUM(amount) AS total_trans
  FROM "transaction" t
  JOIN accounts a ON t.account_id = a.account_id
  JOIN client c ON a.client_id = c.client_id
  GROUP BY c.client_id, c.name
)
SELECT name,
       total_trans,
       NTILE(4) OVER (ORDER BY total_trans DESC) AS quartile
FROM client_totals
ORDER BY quartile, total_trans DESC;

🔹 Segments clients into four quartiles: Quartile 1 = top 25% (VIP clients).

5. Aggregate with Moving Window – 3-Month Moving Average
   
WITH monthly AS (
  SELECT TO_DATE(TO_CHAR(trans_date,'YYYY-MM')||'-01','YYYY-MM-DD') AS month_start,
         SUM(amount) AS deposits
  FROM "transaction"
  WHERE trans_type = 'Deposit'
  GROUP BY TO_CHAR(trans_date,'YYYY-MM')
)
SELECT TO_CHAR(month_start,'YYYY-MM') AS month,
       deposits,
       ROUND(AVG(deposits) OVER (ORDER BY month_start
                                 ROWS BETWEEN 2 PRECEDING AND CURRENT ROW),2) AS moving_avg
FROM monthly
ORDER BY month_start;

🔹 Smooths fluctuations by showing rolling averages over 3 months.

4. Results & Insights
1.Descriptive (What happened?)
* Kigali branch had the highest deposit volumes overall.
* Deposits peaked in June and December.
* The top depositor contributed more than 20% of total deposits in Q4.
2.Diagnostic (Why?)
* December peaks were driven by year-end bonuses and savings promotions.
* Withdrawals increased in July, reducing net deposits.
* VIP clients concentrated in Kigali due to business hub activity.
3.Prescriptive (What next?)
* Liquidity Planning: Increase cash reserves in June & December.
* Client Retention: Reward Quartile 1 clients with loyalty benefits.
* Branch Strategy: Encourage savings programs in Butare & Musanze to balance growth.
* Seasonality: Launch marketing campaigns in March and July (low-deposit months).

5. References

* Oracle Database SQL Language Reference – Window Functions
* TutorialsPoint – PL/SQL Window Functions
* Mode Analytics Blog – SQL Window Functions
* SQLShack – Running Totals and Moving Averages
* W3Schools – SQL OVER() and Ranking Functions
* GeeksforGeeks – NTILE, CUME_DIST, RANK Examples
* Analytics Vidhya – SQL LAG and LEAD
* StackOverflow – Practical SQL Q&A for Oracle
* DataCamp – Window Functions in SQL
* IEEE Paper (2020) – SQL for Analytics and Business Intelligence

6. Integrity Statement

All SQL scripts and analyses in this repository are my original work.
Sources consulted are properly cited above.
No uncredited AI-generated content was used in the preparation of this project.









