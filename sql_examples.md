
Sales Manager: "Please give me the top 5 products ranked by sales. I'd like this ranking done for each year"

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
   
   | Order Year | Product Name            | Total Sales Amount | Total Sales Amount1 |
|------------|-------------------------|--------------------|---------------------|
| 2019       | Mountain-100 Black, 44  | 46574.862          | $46,575             |
| 2019       | Mountain-100 Black, 38  | 44549.868          | $44,550             |
| 2019       | Mountain-100 Black, 48  | 40499.88           | $40,500             |
| 2019       | Road-450 Red, 52        | 40240.524          | $40,241             |
| 2019       | Mountain-100 Black, 42  | 32399.904          | $32,400             |
| 2020       | Mountain-100 Black, 38  | 1130072.8768       | $1,130,073          |
| 2020       | Mountain-100 Black, 44  | 1116778.1163       | $1,116,778          |
| 2020       | Mountain-100 Silver, 38 | 1074269.341        | $1,074,269          |
| 2020       | Mountain-100 Black, 42  | 1070448.2786       | $1,070,448          |
| 2020       | Mountain-100 Silver, 42 | 1033495.3005       | $1,033,495          |
| 2021       | Mountain-200 Black, 38  | 1454407.2611       | $1,454,407          |
| 2021       | Mountain-200 Black, 42  | 1344786.0409       | $1,344,786          |
| 2021       | Mountain-200 Silver, 38 | 1163511.6863       | $1,163,512          |
| 2021       | Mountain-200 Silver, 42 | 1147545.7803       | $1,147,546          |
| 2021       | Mountain-200 Silver, 46 | 1143702.0538       | $1,143,702          |
| 2022       | Mountain-200 Black, 38  | 1549274.3094       | $1,549,274          |
| 2022       | Touring-1000 Blue, 60   | 1293129.1043       | $1,293,129          |
| 2022       | Road-350-W Yellow, 48   | 1252251.6581       | $1,252,252          |
| 2022       | Mountain-200 Black, 42  | 1201528.0151       | $1,201,528          |
| 2022       | Touring-1000 Yellow, 60 | 1142403.6693       | $1,142,404          |
   
  [Result](product_sales_rank_by_year.csv)
