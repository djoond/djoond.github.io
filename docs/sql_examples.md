
Sales Manager: "Please give me the top 5 products sold to our resellers ranked by sales amount.
I'd like this ranking done for both 2021 and 2022"

```SQL
  SELECT [Order Year],
         [Product Name],
	 [Total Sales Amount] = FORMAT([Total Sales Amount], '$#,#')
    FROM (SELECT [Order Year] = YEAR(rs.OrderDate)
                 p.[Product Name],
		 [Total Sales Amount] = SUM(rs.SalesAmount),
		 [Rank] = DENSE_RANK() OVER (PARTITION BY YEAR(rs.OrderDate) ORDER BY SUM(rs.SalesAmount) DESC)
	    FROM FactResellerSales AS rs
	   INNER JOIN Product AS p
	         ON rs.ProductKey = p.ProductKey
	   GROUP BY YEAR(rs.OrderDate), p.[Product Name]
	 ) AS ranked_products
   WHERE Rank <= 5
   ORDER BY [Order Year] ASC, [Total Sales Amount] DESC;
```		 
| Order Year | Product Name            | Total Sales Amount |
|------------|-------------------------|-------------------:|
| 2021       | Mountain-200 Black, 38  |         $1,454,407 |
| 2021       | Mountain-200 Black, 42  |         $1,344,786 |
| 2021       | Mountain-200 Silver, 38 |         $1,163,512 |
| 2021       | Mountain-200 Silver, 42 |         $1,147,546 |
| 2021       | Mountain-200 Silver, 46 |         $1,143,702 |
| 2022       | Mountain-200 Black, 38  |         $1,549,274 |
| 2022       | Touring-1000 Blue, 60   |         $1,293,129 |
| 2022       | Road-350-W Yellow, 48   |         $1,252,252 |
| 2022       | Mountain-200 Black, 42  |         $1,201,528 |
| 2022       | Touring-1000 Yellow, 60 |         $1,142,404 |

***
CFO: "What was our average profit per customer in 2022?
Can we compare it to the previous 2 years? Show the years across the columns, please.
Oh, and while you are at it, can you show average profit per order?"

```SQL
WITH
order_data
AS (SELECT Year = YEAR(OrderDate),
	   [Avg Profit per Order] = ROUND(SUM(SalesAmount - TotalProductCost) / COUNT(DISTINCT SalesOrderNumber), 2)
      FROM FactResellerSales
     WHERE YEAR(OrderDate) IN (2020, 2021, 2022)
     GROUP BY YEAR(OrderDate)
   ),
   
customer_data
AS (SELECT Year = YEAR(OrderDate),
	   [Avg Profit per Customer] = ROUND(SUM(SalesAmount - TotalProductCost) / COUNT(DISTINCT ResellerKey), 2)
      FROM FactResellerSales
     WHERE YEAR(OrderDate) IN (2020, 2021, 2022)
     GROUP BY YEAR(OrderDate)
)

SELECT 'Avg Profit per Order' AS Measure,
        [2020], [2021], [2022]
  FROM order_data 
 PIVOT (
         MAX([Avg Profit per Order])
         FOR Year IN ([2020], [2021], [2022])
       ) AS pivot_table
UNION
SELECT 'Avg Profit per Customer',
        [2020], [2021], [2022]
  FROM customer_data 
 PIVOT (
         MAX([Avg Profit per Customer])
	 FOR Year IN ([2020], [2021], [2022])
       ) AS pivot_table;
```
| Measure                 |   2020 |    2021 |     2022 |
|-------------------------|-------:|--------:|---------:|
| Avg Profit per Customer | 117.21 | 2284.04 | -1005.87 |
| Avg Profit per Order    |  38.06 |  717.23 |  -287.98 |

***
