# Credit Card Fraud-Risk-Analytics


# Table of Contents

[ProjectOverview](#ProjectOverview)

[Tool](#Tools)

[Methodology](#Methodology)


# ProjectOverview

A financial institution is experiencing increasing fraudulent credit card activities across multiple transaction channels, merchants, and countries.

The business wants to:

Identify patterns associated with fraudulent transactions
Detect high-risk customer and merchant behaviors
Understand how fraud varies across geography, card types, transaction methods, and customer demographics
Improve fraud monitoring and risk mitigation strategies

Using SQL, the objective is to analyze transaction data and generate actionable business insights that can support fraud prevention and operational decision-making.

# Tools


# Methodology

```sql

--bq1: Fraud overview and financial impact.

SELECT 
	SUM([Transaction_Amount]) AS Total_Fraud_Amount,
	AVG([Transaction_Amount]) AS Average_Transaction_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1;			

--- | total fraud transactions and fraud rate | ---

WITH FR AS ( SELECT 
				COUNT(*) AS Fraud_Count, 
				COUNT(*) * 100.0 / (SELECT COUNT(*) FROM dbo.credit_card_fraud_detection) AS Fraud_Rate
	FROM dbo.credit_card_fraud_detection
	WHERE Fraudulent = 1
) SELECT Fraud_Count, CAST(Fraud_Rate AS DECIMAL (10,2) ) AS Fraud_Rate_Percentage
FROM FR;			


--- | fraud & non-fraud transactions | ---

SELECT 
	COUNT(CASE WHEN [Fraudulent] = 1 THEN 1 ELSE 0 END) AS YF
FROM credit_card_fraud_detection;


--- bq2: High risk transactions methods. Transaction methods with the highest fraud rates and finacial impacts.
SELECT
    Transaction_Method,
    COUNT(CASE WHEN Fraudulent = 1 THEN 1 END) AS Fraud_Count,
    COUNT(*) AS Total_Transactions,
    ROUND(
        COUNT(CASE WHEN Fraudulent = 1 THEN 1 END) * 100.0
        / COUNT(*), 2
    ) AS Fraud_Rate,
    ROUND(
        SUM(CASE WHEN Fraudulent = 1 THEN Transaction_Amount ELSE 0 END), 2
    ) AS Fraud_Amount
FROM dbo.credit_card_fraud_detection
GROUP BY Transaction_Method
ORDER BY Fraud_Rate DESC;


--- bq3: Fraudulent transactions by MC. MC WITH THE HIGHEST NUMBER OF FRAUDULENT TRANSACTIONS AND THEIR FINANCIAL IMPACTS.

SELECT 
	[Merchant_Category],
	ROUND(AVG([Transaction_Amount]), 2) AS Avg_Fraud_Amount_By_MC,
	COUNT(*) AS Total_Fraudulent_by_MC, 
	CAST(COUNT(*) AS DECIMAL (10, 2)) * 100 / 
(SELECT COUNT(*) FROM dbo.credit_card_fraud_detection) AS Fraud_Rate_Percentage_by_MC
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Merchant_Category];

--- bq4 : COUNTRY LEVEL FRAUD ANALYSIS.	

SELECT [Country],
	COUNT(*) AS CountryFraud,
	SUM([Account_Balance] - [Transaction_Amount]) AS Total_Loss_By_Country,
	COUNT(*) * 100.0 / (SELECT COUNT(*) FROM dbo.credit_card_fraud_detection) AS Country_Fraud_Rate
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Country]
ORDER BY CountryFraud DESC;

--- bq5: OVERTIME FRAUD TRENDS.

SELECT 
	COUNT(*) AS Fraudulent_Transactions,
	SUM(COUNT(*)) OVER (ORDER BY YEAR(Transaction_Date), MONTH(Transaction_Date)) AS Cumulative_Fraudulent_Transactions,
	DATENAME(MONTH, Transaction_Date) AS Fraudulent_Transactions_By_Month,
	DATENAME(YEAR, Transaction_Date) AS Fraudulent_Transactions_By_Year
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY 
	YEAR(Transaction_Date),
	MONTH(Transaction_Date),
	DATENAME(MONTH, Transaction_Date),
	DATENAME(YEAR, Transaction_Date)
ORDER BY YEAR(Transaction_Date), MONTH(Transaction_Date);--Fraudulent_Transactions_By_Day;


-- bq6: Which card types are most vulnerable to frudulent transactions?

SELECT COUNT(*) AS Fraudulent_Transactions_By_Card_Type, 
	[Card_Type],
	ROUND(SUM([Transaction_Amount]), 2) AS Fraud_Amount_By_Card_Type,
	CAST(COUNT(*) AS DECIMAL (10, 2)) * 100 / (SELECT COUNT(*) FROM dbo.credit_card_fraud_detection)   AS Fraud_Rate_Percentage_By_Card_Type
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Card_Type]
ORDER BY Fraudulent_Transactions_By_Card_Type DESC;

-- bq7: Which customer demographics are associated with higher fraud exposure?

SELECT 
    [Country],
    [Transaction_Location],
    [User_Gender],
    CASE 
        WHEN [User_Age] BETWEEN 18 AND 25 THEN 'Under 25'
        WHEN [User_Age] BETWEEN 26 AND 35 THEN 'Young Age'
        WHEN [User_Age] BETWEEN 36 AND 45 THEN 'Middle Age'
        WHEN [User_Age] BETWEEN 46 AND 65 THEN 'Old Age'
        ELSE 'Very Old Age'
        END AS Age_Group,
    COUNT(*) AS Fraudulent_Transactions,
    ROUND(SUM([Transaction_Amount]), 2) AS Total_Fraud_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Country], [Transaction_Location],[User_Gender],
         CASE 
            WHEN [User_Age] BETWEEN 18 AND 25 THEN 'Under 25'
            WHEN [User_Age] BETWEEN 26 AND 35 THEN 'Young Age'
            WHEN [User_Age] BETWEEN 36 AND 45 THEN 'Middle Age'
            WHEN [User_Age] BETWEEN 46 AND 65 THEN 'Old Age'
            ELSE 'Very Old Age'
         END;


-- bq8: Do fraudulent transactions tend to involve higher transaction amounts?

SELECT      
        'Fraud' AS Metric,
        ROUND(SUM([Transaction_Amount]), 2) AS Total_Distribution_Amount,
        ROUND(AVG([Transaction_Amount]), 2) AS Average_Transaction_Amount,
        ROUND(MAX([Transaction_Amount]), 2) AS Max_Transaction_Amount,
        ROUND(MIN([Transaction_Amount]), 2) AS Min_Transaction_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1

UNION ALL 

SELECT      
        'Normal' AS Metric,
         ROUND(SUM([Transaction_Amount]), 2) AS Total_Distribution_Amount,
         ROUND(AVG([Transaction_Amount]), 2) AS Average_Transaction_Amount,
         ROUND(MAX([Transaction_Amount]), 2) AS Max_Transaction_Amount,
         ROUND(MIN([Transaction_Amount]), 2) AS Min_Transaction_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 0;

 
 -- high value transactions.

SELECT                                                     
         [Country], 
         'High Value' AS Transaction_Status,
         ROUND(SUM([Transaction_Amount]), 2) AS Total_Fraud_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1 AND [Transaction_Amount] >= 800
GROUP BY [Country]

                                UNION ALL

SELECT 
        [Country], 
        'Mid Value' AS Transaction_Status,
        ROUND(SUM([Transaction_Amount]), 2) AS Total_Fraud_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1 AND [Transaction_Amount] BETWEEN 600 AND 799
GROUP BY [Country]

                                UNION ALL

SELECT 
        [Country], 
        'Low Value' AS Transaction_Status,
        ROUND(SUM([Transaction_Amount]), 2) AS Total_Fraud_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1 AND [Transaction_Amount] < 600
GROUP BY [Country];


-- bq9: Which merchant generates the highest of fraudulent transactions & loss?

SELECT  
       [Merchant_Name],
       COUNT(*) AS Highest_Fraud_Transactions, 
       ROUND(SUM([Transaction_Amount]), 2) AS Fraud_Loss_Amount_By_Merchant
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Merchant_Name]
ORDER BY Highest_Fraud_Transactions DESC; 


-- bq10: Is there a relationship between account balance and fraudulent transactions? 


SELECT [Merchant_Name],                                         
CASE 
    WHEN [Account_Balance] < 1000 THEN 'Low Balance'
    WHEN [Account_Balance] BETWEEN 1000 AND 5000 THEN 'Medium Balance'
    ELSE 'High Balance'
END AS Account_Balance_Category,
COUNT(*) AS Fraudulent_Transactions,
SUM([Transaction_Amount]) AS Total_Fraud_Amount
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Merchant_Name], 
         CASE 
             WHEN [Account_Balance] < 1000 THEN 'Low Balance'
             WHEN [Account_Balance] BETWEEN 1000 AND 5000 THEN 'Medium Balance'
             ELSE 'High Balance'
         END;                                          

 -- average account balance for fraud victims.

SELECT                                          
    CAST(AVG([Account_Balance]) AS DECIMAL(10,2)) AS Average_Account_Balance_For_Fraudulent_Transactions
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Merchant_Name];                              



-- bq11: Are there specific times of day when fraudulent transactions are more likely to occur?

WITH T1 AS ( SELECT 
    CASE 
        WHEN CAST(DATENAME(HOUR, [Transaction_Time]) AS INT) >= 22 
            OR CAST(DATENAME(HOUR, [Transaction_Time]) AS INT) <= 4 THEN 'Late Night'
        WHEN CAST(DATENAME(HOUR, [Transaction_Time]) AS INT) BETWEEN 19 AND 21 THEN 'Night'
        WHEN CAST(DATENAME(HOUR, [Transaction_Time]) AS INT) BETWEEN 16 AND 18 THEN 'Evening'
        WHEN CAST(DATENAME(HOUR, [Transaction_Time]) AS INT) BETWEEN 12 AND 15 THEN 'Afternoon'
        WHEN CAST(DATENAME(HOUR, [Transaction_Time]) AS INT) BETWEEN 7 AND 11 THEN 'Morning'
        ELSE 'Early Morning'
        END AS Transaction_Hour,
        [Transaction_Amount],
        Transaction_ID,
        Fraudulent 
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1 ), 

T2 AS ( SELECT  Transaction_Hour, 
                COUNT(*) AS Fraudulent_Transactions,
                ROUND(SUM([Transaction_Amount]), 2) AS Total_Fraud_Amount 
                FROM  T1
                GROUP BY Transaction_Hour )

SELECT Transaction_Hour, Fraudulent_Transactions, Total_Fraud_Amount
FROM T2
ORDER BY Fraudulent_Transactions DESC;


--bq12: which transaction locations are associated with the highest fraud occurrence?

SELECT [Country], [Transaction_Location],
        COUNT(*) AS Fraudulent_Locations,
        ROUND(SUM([Transaction_Amount]), 2) AS Total_Fraud_Amount_By_Location
FROM dbo.credit_card_fraud_detection
WHERE Fraudulent = 1
GROUP BY [Country], [Transaction_Location]
ORDER BY Fraudulent_Locations DESC;


```sql
