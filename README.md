# Northwind Report - PowerBI Project

Hello everyone,

Thanks for reaching out my repository.

Here you can find the report:

https://www.novypro.com/project/northwind-traders-report---portfolio-project-power-bi

Please see the description below, which include more details about the project and analyze the data.


# Introduce

The report is based on a free data base "Northwind" and has been created for non-commercial purposes to show my skills as a Power BI Analyst. It is dedicated to help the business unit manage, monitor major KPI and maintain the current processes in the company.

The database contains the sales data for Northwind Traders, a fictitious specialty foods export­/import company.

# Data Model

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/8e3b6d8e-ec82-4be8-81b6-bf3840fe5601)

The model includes couple additional tables:

**factPromoPL** 

**factShipLevelPL**

To make the report more realistic, I have created fictitious budget plans to allow better comparision on these two KPI's. For the Promotion in % it's set for 7% for each month and for the Ship Level it's set for 95% for each month.

**factShipRealization** table includes information about the current realization time of orders, required order date, deviation from the PL and general information if the order is delayed or not. 

The table was created with the following SQL query:


    SELECT O.OrderID
    	,DATEDIFF(DAY, O.OrderDate, O.ShippedDate) ShipDaysAC
    	,DATEDIFF(DAY, o.OrderDate, O.RequiredDate) ShipDaysPL
    	,DATEDIFF(DAY, DATEDIFF(DAY, o.OrderDate, O.RequiredDate), DATEDIFF(DAY, O.OrderDate, O.ShippedDate)) 'PL Deviation'
    	,CASE
    	WHEN DATEDIFF(DAY, DATEDIFF(DAY, O.OrderDate, O.ShippedDate), DATEDIFF(DAY, o.OrderDate, O.RequiredDate)) < 0 THEN 1
    	WHEN DATEDIFF(DAY, DATEDIFF(DAY, O.OrderDate, O.ShippedDate), DATEDIFF(DAY, o.OrderDate, O.RequiredDate)) IS NULL THEN NULL
    	ELSE 0
    	END 'Is delayed?'
    FROM 
    	Orders O


**dimDate** is a calendar table which has been created using DAX Language:

    DimDate = 
    ADDCOLUMNS (
    CALENDAR ("1996-01-01", "1999-12-31"),
    "DateInt", FORMAT ( [Date], "YYYYMMDD" ),
    "Year", YEAR ( [Date] ),
    "Monthnumber", FORMAT ( [Date], "MM" ),
    "MonthNameShort", FORMAT ( [Date], "mmm", "en-US" ),
    "MonthNameLong", FORMAT ( [Date], "mmmm" , "en-US"),
    "DayOfWeekNumber", WEEKDAY ( [Date] ),
    "Week Number", WEEKNUM([Date]),
    "DayOfWeek", FORMAT ( [Date], "dddd" ),
    "DayOfWeekShort", FORMAT ( [Date], "ddd" ),
    "Quarter", "Q" & FORMAT ( [Date], "Q" ),
    "YearQuarter", FORMAT ( [Date], "YYYY" ) & "/Q" & FORMAT ( [Date], "Q" )
    )



# Calculations correctness check - DAX vs SQL

Here you can find major KPI calculated in the MS SQL Server compared to the calculations created in Power BI:

We will focus on following KPI's:

**Sales AC** - total current sales with the discounts included

**Orders AC** - total number of orders

**Ship % AC** - percentage of ships realized on-time

**Promo % AC** - percentage share of the promotion in the total sales


SQL Calculations:


    WITH ShipAC as (
      SELECT 
        O.OrderID, 
        DATEDIFF(DAY, O.OrderDate, O.ShippedDate) ShipDaysAC, 
        DATEDIFF(DAY, o.OrderDate, O.RequiredDate) ShipDaysPL, 
        DATEDIFF(
          DAY, 
          DATEDIFF(DAY, o.OrderDate, O.RequiredDate), 
          DATEDIFF(DAY, O.OrderDate, O.ShippedDate)
        ) Deviation, 
        CASE WHEN DATEDIFF(
          DAY, 
          DATEDIFF(DAY, O.OrderDate, O.ShippedDate), 
          DATEDIFF(DAY, o.OrderDate, O.RequiredDate)
        ) < 0 THEN 1 WHEN DATEDIFF(
          DAY, 
          DATEDIFF(DAY, O.OrderDate, O.ShippedDate), 
          DATEDIFF(DAY, o.OrderDate, O.RequiredDate)
        ) IS NULL THEN NULL ELSE 0 END 'Is delayed?' 
      FROM 
        Orders o
    ), 
    PromoAC as (
      SELECT 
        YEAR(o.OrderDate) 'Year', 
        SUM(
          od.Quantity * od.UnitPrice * od.Discount
        ) Promo, 
        (
          SELECT 
            SUM(
              od2.Quantity * od2.UnitPrice * (1 - od2.Discount)
            ) 
          FROM 
            [Order Details] od2 
            JOIN Orders o2 on o2.OrderID = od2.OrderID 
          WHERE 
            YEAR(o.OrderDate) = YEAR(o2.OrderDate)
        ) total_sales 
      FROM 
        [Order Details] od 
        JOIN Orders o on OD.OrderID = O.OrderID 
      WHERE 
        od.Discount <> 0 
      GROUP BY 
        YEAR(o.OrderDate)
    ), 
    SalesAC as (
      SELECT 
        YEAR(o3.OrderDate) 'Year', 
        SUM(
          od3.Quantity * od3.UnitPrice * (1 - od3.Discount)
        ) 'Sales AC' 
      FROM 
        [Order Details] od3 
        JOIN Orders o3 on o3.OrderID = od3.OrderID 
      WHERE 
        YEAR(o3.OrderDate) = YEAR(o3.OrderDate) 
      GROUP BY 
        YEAR(o3.OrderDate)
    ), 
    OrdersAC as (
      SELECT 
        YEAR(o4.OrderDate) 'Year', 
        COUNT(DISTINCT od4.OrderID) 'Orders AC' 
      FROM 
        [Order Details] od4 
        JOIN Orders o4 on o4.OrderID = od4.OrderID 
      GROUP BY 
        YEAR(o4.OrderDate)
    ) 
    SELECT 
      YEAR(o.OrderDate) 'Year', 
      FORMAT(
        MAX([Sales AC]), 
        '### ##0.00'
      ) 'Sales AC', 
      COUNT([Orders AC]) 'Orders AC', 
      FORMAT(
        (
          SUM([Is delayed?]) * 1.0 / count(ShipAC.OrderID) -1
        ) * -1, 
        'P2'
      ) 'Ship AC', 
      FORMAT(
        MAX(Promo / total_sales), 
        'P2'
      ) 'Promo AC' 
    FROM 
      ShipAC 
      JOIN Orders o on ShipAC.OrderID = o.OrderID 
      JOIN PromoAC on PromoAC.Year = YEAR(o.OrderDate) 
      JOIN SalesAC on SalesAC.Year = YEAR(o.OrderDate) 
      JOIN OrdersAC on OrdersAC.Year = YEAR(o.OrderDate) 
    GROUP BY 
      YEAR(o.OrderDate) 
    ORDER BY 
      1 ASC;


**Output:**

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/d0e0c1e3-ba36-4104-8e84-ae40ee75424b)


PowerBI Calculations:

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/d0bcc717-c403-4448-b810-52262063f1af)


A full description of the DAX metrics used can be found on the last page of the report.


# Tab description


**Sales Cockpit** - This is general dashboard providing aggregated data on different level. You can choose from if you want to see results for every Category, Country, Customers, Employee, Products or Suppliers. Additionally, you can switch the metrics and you can choose between sum and average

**Shippers** - Main purpose of this tab is monitoring accuracy of the ships and how exactly all three companies were performing during the year

**Orders** - Orders master data, you can see full information about each order, e.g. order date, employee involved or realization time

**Stock Management** - The tab was created to help the business unit monitor current stocks amounts, you can see red highlited items that are overstocked on the warehouse and also information about the last date the product has been ordered

**Promotion History** - You can follow the history of discounts for each article during the year. To make the promotion history easier to manage, there were created four discounts group which has been defined on the right side of the page

**Measures Catalog** - description of every measure that has been created and used in this report


# Custom SQL analyze - year 1997

In this part, we will perform a simple analyze using SQL language. At first, we will take a look on employees and their productivity:

**Employee productivity:**

        SELECT Concat(e.firstname, ' ', e.lastname) AS Employee,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 1 THEN o.orderid
                              END)                  January,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 2 THEN o.orderid
                              END)                  February,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 3 THEN o.orderid
                              END)                  March,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 4 THEN o.orderid
                              END)                  April,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 5 THEN o.orderid
                              END)                  May,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 6 THEN o.orderid
                              END)                  June,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 7 THEN o.orderid
                              END)                  July,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 8 THEN o.orderid
                              END)                  August,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 9 THEN o.orderid
                              END)                  September,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 10 THEN o.orderid
                              END)                  October,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 11 THEN o.orderid
                              END)                  November,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) = 12 THEN o.orderid
                              END)                  December,
               Count(DISTINCT CASE
                                WHEN Month(o.orderdate) BETWEEN 1 AND 12 THEN o.orderid
                              END)                  Total
        FROM   orders o
               JOIN [order details] od
                 ON o.orderid = od.orderid
               JOIN employees e
                 ON o.employeeid = e.employeeid
        WHERE  Year(o.orderdate) = 1997
        GROUP  BY Concat(e.firstname, ' ', e.lastname)
        ORDER  BY 14 DESC; 


**Output:**

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/e50439c9-df77-449e-8a23-a34d8e504787)


As we can see, Margaret Peacock was the one with the most orders completed in the whole year. Now let's check, whose orders earned the biggest revenue for the company:


        WITH employeeanalyze
             AS (SELECT Concat(e.firstname, ' ', e.lastname)                  Employee,
                        Sum(od.quantity * od.unitprice * ( 1 - od.discount )) Sales,
                        Avg(od.quantity * od.unitprice * ( 1 - od.discount )) AvgSales,
                        (SELECT Avg(od.quantity * od.unitprice * ( 1 - od.discount ))
                         FROM   [order details] od
                                JOIN orders o
                                  ON o.orderid = od.orderid
                         WHERE  Year(o.orderdate) = 1997)
                        'TotalAvgSales'
                 FROM   employees e
                        JOIN orders o
                          ON o.employeeid = e.employeeid
                        JOIN [order details] od
                          ON od.orderid = o.orderid
                 WHERE  Year(o.orderdate) = 1997
                 GROUP  BY Concat(e.firstname, ' ', e.lastname))
        SELECT employee,
               Format(sales, '### ###')         Sales,
               Format(avgsales, '### ###')      AvgSales,
               Format(totalavgsales, '### ###') 'TotalAvgSales',
               CASE
                 WHEN Round(avgsales, 0) > Round(totalavgsales, 0) THEN '+'
                 WHEN Round(avgsales, 0) = Round(totalavgsales, 0) THEN '='
                 ELSE '-'
               END                              AS Deviation
        FROM   employeeanalyze
        ORDER  BY Cast(sales AS INT) DESC; 


**Output:**

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/fa4831b8-661a-4ba4-b53d-11d515a0012d)


Again, we can see that Margaret Peacock is way ahead comparing to the rest of employees in total sales, but very curious is a thing, that she has not the best average sales.  Let's see, how she performed in that case during the year on a bar chart:

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/899dd4d0-eb40-4137-8312-778ffea5d60e)


Margaret Peacock was above the average for total company only in 5 months, which means she was below 50%. We can see a strong peak in Januray. Let's see, what happened in that period:


        SELECT Concat(e.firstname, ' ', e.lastname)
               Employee,
               o.orderid,
               Count(od.productid)
               '# Items',
               Format(Sum(od.quantity * od.unitprice * ( 1 - od.discount )), '### ###')
               Sales
        FROM   [order details] od
               JOIN orders o
                 ON od.orderid = o.orderid
               JOIN employees e
                 ON o.employeeid = e.employeeid
        WHERE  Year(o.orderdate) = 1997
               AND Month(o.orderdate) = 1
        GROUP  BY Concat(e.firstname, ' ', e.lastname),
                  o.orderid
        ORDER  BY 1 DESC; 


**Output:**


![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/1315ca45-6e5e-4888-bc1d-74b39afc614a)


We can see an order **NO 10417**, which definitely increased the average for Mrs. Peacock. Let's get deeper into that order:


        SELECT o.orderid,
               p.productname,
               od.quantity,
               od.unitprice
        FROM   [order details] od
               JOIN orders o
                 ON od.orderid = o.orderid
               JOIN employees e
                 ON o.employeeid = e.employeeid
               JOIN products p
                 ON od.productid = p.productid
        WHERE  o.orderid = 10417
        ORDER  BY 3 DESC; 


**Output:**


![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/35e63f2a-79b9-463e-afd6-f6920dd2a7b8)


The peak was caused a big order (50qty) on an expensive product named 'Côte de Blaye' and it had a significant impact on a monthly average result.


But don't remember that number of orders may have a big impact on the total sales value. Let's check if there exists any correlation:

        WITH correlation
             AS (SELECT Concat(E.firstname, ' ', e.lastname)                  Employee,
                        Sum(od.quantity * od.unitprice * ( 1 - od.discount )) Sales,
                        Count(DISTINCT OD.orderid)                            Orders
                 FROM   [order details] od
                        JOIN orders o
                          ON od.orderid = o.orderid
                        JOIN employees e
                          ON o.employeeid = e.employeeid
                 WHERE  Year(O.orderdate) = 1997
                 GROUP  BY Concat(E.firstname, ' ', e.lastname))
        SELECT Round(( Avg(sales * orders) - ( Avg(sales) * Avg(orders) ) ) / (
                            Stdevp(sales) * Stdevp(orders) ), 2)
               'Pearson Linear Correlation'
        FROM   correlation 


**Output:**

![image](https://github.com/michalpugaczew/PowerBI-Project/assets/152793313/24f2baab-4beb-47d9-890d-75da974e429f)


We can see that there is strong, significant correlation between number of orders and total sales. This means that as the number of orders increases, total sales also increase and that explains the good result of one of the employees, mrs Peacock.


**A few words in conclusion**


Thank you for reading the entire descritpion. The purpose of the project was about showing sample of my skills as  a Power BI Analyst. If you are interested in cooperation with me or you have any question, you can contact with me via my LinkedIn profile:

https://www.linkedin.com/in/micha%C5%82-pugaczew-a963221a3/
