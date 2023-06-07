
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

CFO: "What was our average profit per reseller customer in 2022?
Can we compare it to the previous 2 years? Show the years across the columns, please.
Oh, and while you are at it, can you show average profit per order for resellers?"

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

Sales Manager: "Can you please tell me, by month, the count of online orders in 2022 that included only a bike
(or bikes) purchase versus a count of orders for which no bike was ordered as well as a count of mixed category orders?
What percentage of our total orders are "bike only"?

```SQL
DROP TABLE IF EXISTS #sales_orders_including_bikes;

SELECT DISTINCT SalesOrderNumber,
       [Order Month] = CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, OrderDate),0) AS date)
  INTO #sales_orders_including_bikes 
  FROM FactInternetSales
 WHERE ProductKey IN (SELECT ProductKey FROM DimProduct AS p
                             INNER JOIN DimProductSubcategory AS ps
	                     ON p.ProductSubcategoryKey = ps.ProductSubcategoryKey
			     INNER JOIN DimProductCategory AS pc
	                     ON ps.ProductCategoryKey = pc.ProductCategoryKey
                       WHERE EnglishProductCategoryName = 'Bikes');

DROP TABLE IF EXISTS #sales_orders_including_non_bikes;

SELECT DISTINCT SalesOrderNumber,
       [Order Month] = CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, OrderDate),0) AS date)
  INTO #sales_orders_including_non_bikes
  FROM FactInternetSales
 WHERE ProductKey IN (SELECT ProductKey FROM DimProduct AS p
                             INNER JOIN DimProductSubcategory AS ps
	                     ON p.ProductSubcategoryKey = ps.ProductSubcategoryKey
			     INNER JOIN DimProductCategory AS pc
	                     ON ps.ProductCategoryKey = pc.ProductCategoryKey
                       WHERE EnglishProductCategoryName <> 'Bikes');

-- only bike category order counts
WITH
bike_only
AS (SELECT [Order Month],
	   [Order Count] = COUNT(SalesOrderNumber)
      FROM (SELECT SalesOrderNumber, [Order Month] FROM #sales_orders_including_bikes
            EXCEPT
            SELECT SalesOrderNumber, [Order Month] FROM #sales_orders_including_non_bikes) AS o
      GROUP BY [Order Month]
   ),

-- only non-bike category order counts
non_bike_only
AS (SELECT [Order Month],
	   [Order Count] = COUNT(SalesOrderNumber)
      FROM (SELECT SalesOrderNumber, [Order Month] FROM #sales_orders_including_non_bikes
            EXCEPT
            SELECT SalesOrderNumber, [Order Month] FROM #sales_orders_including_bikes) AS o
      GROUP BY [Order Month]
   ),

-- mixed orders
mixed
AS (SELECT [Order Month],
	   [Order Count] = COUNT(SalesOrderNumber)
      FROM (SELECT SalesOrderNumber, [Order Month] FROM #sales_orders_including_non_bikes
            INTERSECT
            SELECT SalesOrderNumber, [Order Month] FROM #sales_orders_including_bikes) AS o
      GROUP BY [Order Month]
   )

 SELECT DISTINCT [Order Month] = CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, s.OrderDate),0) AS date), 
        [Bike Category Only Orders] = b.[Order Count],
        [Non-bike Categories Only Orders] = nb.[Order Count],
        [Mixed Orders] = m.[Order Count],
	[Total Orders] = b.[Order Count] + nb.[Order Count] + m.[Order Count],
	[Percent of Bike-Only Orders] =  FORMAT(CAST(b.[Order Count] AS float) /
	                                 (b.[Order Count] + nb.[Order Count] + m.[Order Count]), 'P')
   FROM FactInternetSales AS s
        LEFT OUTER JOIN bike_only AS b
	ON DATEADD(MONTH, DATEDIFF(MONTH, 0, s.OrderDate),0) = b.[Order Month]
        LEFT OUTER JOIN non_bike_only AS nb
	ON DATEADD(MONTH, DATEDIFF(MONTH, 0, s.OrderDate),0) = nb.[Order Month]
        LEFT OUTER JOIN mixed AS m
	ON DATEADD(MONTH, DATEDIFF(MONTH, 0, s.OrderDate),0) = m.[Order Month]
  WHERE YEAR(s.OrderDate) = '2022'
  ORDER BY [Order Month] ASC;
  ```
| Order Month | Bike Category Only Orders | Non-bike Categories Only Orders | Mixed Orders | Total Orders | Percent of Bike-Only Orders |
|-------------|--------------------------:|--------------------------------:|-------------:|-------------:|----------------------------:|
| 2022-01-01  |                        65 |                             135 |          432 |          632 |                      10.28% |
| 2022-02-01  |                        53 |                             967 |          401 |         1421 |                       3.73% |
| 2022-03-01  |                        83 |                            1082 |          525 |         1690 |                       4.91% |
| 2022-04-01  |                        68 |                            1008 |          536 |         1612 |                       4.22% |
| 2022-05-01  |                       100 |                            1043 |          649 |         1792 |                       5.58% |
| 2022-06-01  |                       136 |                            1003 |          868 |         2007 |                       6.78% |
| 2022-07-01  |                        99 |                            1078 |          698 |         1875 |                       5.28% |
| 2022-08-01  |                        97 |                            1069 |          800 |         1966 |                       4.93% |
| 2022-09-01  |                       103 |                            1014 |          767 |         1884 |                       5.47% |
| 2022-10-01  |                       133 |                            1115 |          883 |         2131 |                       6.24% |
| 2022-11-01  |                       127 |                            1012 |          948 |         2087 |                       6.09% |
| 2022-12-01  |                       141 |                            1057 |          994 |         2192 |                       6.43% |

***
Sales Manager: "Please create a monthly count of newly acquired reseller customers for 2021-2022"

```SQL
WITH
existing_customers
AS (SELECT DISTINCT ResellerKey
      FROM FactResellerSales
     WHERE OrderDate < '2021-01-01'
   ),
/* customers considered acquired on the date of their first order */
initial_order_dates
AS (SELECT ResellerKey,
           [Initial Order Date] = MIN(OrderDate)
      FROM FactResellerSales
     WHERE ResellerKey NOT IN (SELECT ResellerKey FROM existing_customers)
     GROUP BY ResellerKey
   )

SELECT [Order Month] = CAST(DATEADD(MONTH, DATEDIFF(MONTH, 0, OrderDate),0) AS date),
       [New Resellers] = COUNT(DISTINCT i.ResellerKey)
  FROM FactResellerSales AS s
       LEFT OUTER JOIN initial_order_dates AS i
       ON DATEADD(MONTH, DATEDIFF(MONTH, 0, s.OrderDate),0) =
       DATEADD(MONTH, DATEDIFF(MONTH, 0, [Initial Order Date]),0)
 WHERE s.OrderDate >= '2021-01-01'
 GROUP BY DATEADD(MONTH, DATEDIFF(MONTH, 0, s.OrderDate),0)
 ORDER BY [Order Month] ASC
 ```
| Order Month | New Resellers |
|-------------|--------------:|
| 2021-01-01  |            79 |
| 2021-02-01  |            71 |
| 2021-03-01  |             3 |
| 2021-04-01  |             7 |
| 2021-05-01  |             8 |
| 2021-06-01  |             0 |
| 2021-07-01  |             3 |
| 2021-08-01  |             1 |
| 2021-09-01  |             1 |
| 2021-10-01  |             1 |
| 2021-11-01  |             2 |
| 2021-12-01  |            35 |
| 2022-01-01  |            72 |
| 2022-02-01  |            88 |
| 2022-03-01  |             4 |
| 2022-04-01  |             1 |
| 2022-05-01  |             2 |
| 2022-06-01  |             0 |
| 2022-07-01  |             1 |
| 2022-08-01  |             1 |
| 2022-09-01  |             0 |
| 2022-10-01  |             0 |
| 2022-11-01  |             3 |

***
