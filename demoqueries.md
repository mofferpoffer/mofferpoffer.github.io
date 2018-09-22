# Demo Queries
## 1. Total Revenue

**Calculate revenue of all registered sales transactions.**

In order to properly value the benefits of having crated a dimensional model of the contents of a transactional model, we first show solutions for this query on both the transactional and the dimensional database.

### On transactional db CustomerSales

The idea is to first calculate the value of each order line and then sum up those values but we first have to retrieve the price a product had when the order was placed.

```sql
select sum(quantity * price) as revenue
from custorder o
join orderline l
    on o.orderno = l.orderno
join productprice p
    on p.prodno = l.prodno
and orderdate between startdate and enddate
;
```
### On dimensional db CustomerSalesDW

As the dimensional database is for reporting and analysis purposes only (we won't use it to change data as for instance changing a product's price), we've already calculated the proper order line value upon creating the database. This makes many of our (reporting) queries so much easier!

```sql
select sum(linetotal) as revenue
from factSales
;
```

## 2. Revenue per region

**Calculate revenue per sales region of all registered sales transactions.**

### On transactional db CustomerSales

In order to get each customer's region code, you have to look it up in the sales region table using part of the customer address as lookup key:

```sql
select regioncode, sum(quantity * price) as revenue
from salesregion r
join customer c
    on left(address, 4) between pcbegin and pcend
join custorder o
    on c.custno = o.custno
join orderline l
    on o.orderno = l.orderno
join productprice p
    on p.prodno = l.prodno
and orderdate between startdate and enddate
group by regioncode
;
```

### On dimensional db CustomerSalesDW

In the dimensional version of CustomerSales, the Customer and SalesRegion tables have been flattened into a single dimDimension table. This leads to much easier retrieval:

```sql
select regioncode, sum(linetotal) as revenue
from factSales s
join dimCustomer c
    on s.custno = c.custno
group by regioncode
;
```

## 3. Monthly revenue

**Calculate monthly revenue per product category in 2014.**

### On transactional db CustomerSales

```sql
select month(orderdate) as monthno, catcode, sum(quantity * price) as revenue
from custorder o
join orderline l
    on o.orderno = l.orderno
join product p
    on p.prodno = l.prodno
join productprice pp
    on pp.prodno = l.prodno
and orderdate between startdate and enddate
where orderdate between '2014-01-01' and '2014-12-31'
group by month(orderdate), catcode
order by month(orderdate), catcode
;
```

### On dimensional db CustomerSalesDW

```sql
select monthno, monthname, catcode, sum(linetotal) as revenue
from factSales s
join dimDate
    on orderdate = dat
join dimProduct p
    on p.prodno = s.prodno
where year = 2014
group by monthno, monthname, catcode
;
```

## 4. Sales transactions per month

**Calculate for each customer the number of sales transactions per month in 2014. Show months without transactions as 0.**

I hope I have convincingly made my case that it is easier to query a dimensional database than a transactional database. All remaining queries are defined on the dimensional database.

In one of the earlier queries, months without sales for any of the product categories, wouldn't be visible. For reporting purposes, this is often undesirable. Instead of being invisible, months without sales should be presented as months with 0 revenue. Likewise, months without transactions should be presented as months with 0 transactions.

Using a standard inner join won't do the job: it leaves out months without transactions:

```sql
-- wrong result!
select month(orderdate) as monthno, c.custno, name, count(distinct orderno) as [#transactions]
from dimCustomer c
join factSales s
    on s.custno = c.custno
where year(orderdate) = 2014
group by month(orderdate), c.custno, name
order by name, monthno
;
```

Note that factSales has order line granularity. In order to get the number of transactions, we have to count the number of unique (distinct) order numbers.

We have to use an outer join to preserve month without sales. As a result, we can't use the orderdate from factSales and that's exactly the reason we've created a date dimension in our model. A first (wrong) attempt could be:

```sql
-- wrong result!
select monthno, c.custno, name, count(distinct orderno) as [#transactions]
from dimCustomer c
left join factSales s
    on s.custno = c.custno
left join dimDate
    on orderdate = dat
where year = 2014
group by monthno, c.custno, name
order by name, monthno
;
```

Run the query and explain why this is not working.

The following query is a working solution. It cross joins dimCustomer with dimDate in order to create all possible customer-month combinations. The result of the cross join is the (left) outer joined with factSales and thus effectively preserve all customer-month combination without sales transactions.

```sql
select monthno, c.custno, name, count(distinct orderno) as [#transactions]
from dimCustomer c
cross join dimDate
left join factSales s
    on s.custno = c.custno
    and orderdate = dat
where year = 2014
group by monthno, c.custno, name
order by name, monthno
;
```

## 5. Monthly product revenue

**Calculate monthly revenue of zwezerik (product: sweetbread) in 2014. Show months without sales as 0.**

A working solution in the spirit of previous solution would be:

```sql
select monthno, monthname, isnull(sum(linetotal), 0) as revenue
from dimDate
cross join dimProduct p
left join factSales s
    on p.prodno = s.prodno
    and orderdate = dat
where year = 2014
and proddesc = 'zwezerik'
group by monthno, monthname
order by monthno
;
```

That's it, working. My advise would be to stick to this pattern of solving this kind of questions.

If you're interested, an alternative solution would be:

```sql
select monthno, monthname, isnull(sum(linetotal), 0) as revenue
from factSales s
join dimProduct p
    on p.prodno = s.prodno
    and proddesc = 'zwezerik'
right join dimDate
    on orderdate = dat
where year = 2014
group by monthno, monthname
order by monthno
;
```

This is an interesting query. Why did we use a right join instead of the more common left join? Let's set up a solution using left joins:

```sql
-- wrong result!
select monthno, monthname, isnull(sum(linetotal), 0) as revenue
from dimDate
left join factSales s
    on orderdate = dat
left join dimProduct p
    on p.prodno = s.prodno
    and proddesc = 'zwezerik'
where year = 2014
group by monthno, monthname
order by monthno
;
```

As joins evaluate from left to right, we obviously have to left join dimProduct as well in order to keep preserving the empty months from dimDate. However, the query does not return the correct results. Analyzing the results will learn you that the filter on proddesc (selecting 'zwezerik' only) hasn't worked out. The problem is that dimProduct is left joined with the result of the first left join. This means that all factSales lines will be in the result set even if there were no 'zwezerik' sales involved. And as line totals are summed from the factSales table, this obviously leads to the wrong result. As a simple insight in the problem, you could add a count on linetotal and a count on proddesc:

```sql
-- wrong result!
select monthno, monthname, isnull(sum(linetotal), 0) as revenue,
    count(linetotal) as [#linetotal values], count(proddesc) as [#zwezerik lines]
from dimDate
left join factSales s
    on orderdate = dat
left join dimProduct p
    on p.prodno = s.prodno
    and proddesc = 'zwezerik'
where year = 2014
group by monthno, monthname
order by monthno
;
```

Having different values for the number of factsales lines and order lines that involved 'zwezerik', means that you are going to end up with the wrong result. These mistakes are very difficult to spot as you get a seemingly proper result. It really demands a good understanding of SQL to avoid such mistakes!

It is now easy to understand that a solution with a cross join or a right join as above is preferable over a left join solution. Here is a working left join solution:

```sql
select monthno, monthname, isnull(sum(linetotal), 0) as revenue
from dimDate
left join (
    factSales s
    join dimProduct p
        on p.prodno = s.prodno
        and proddesc = 'zwezerik'
)
    on orderdate = dat
where year = 2014
group by monthno, monthname
order by monthno
;
```

Create a column chart of the result in Excel.

Here' s a short digression about creating Excel charts from SQL query results.

To create an Excel visual of the result of this query, we use the following procedure:

1. Create a database connection to the SQL Server database using Power Query (Get & Transform).
2. Open Advanced Options in the connection dialog and past your tested T-SQL query.
3. Adapt your query such that the data series you want to visualize are in separate columns. (We often refer to this format as pivot format.)
4. Load the query results to a table in a new worksheet.
5. Use this table as a chart feeder for a suitable Excel chart.

Why not using a pivot table with pivot charts? First of all a pivot table with pivot charts is a fine solution and indeed much more common than executing a query on the database for each visual. However, pivot tables (and I mean traditional pivot tables, not Power Pivot pivot tables) come with their disadvantages:

1. not all available chart types are available as pivot chart.
2. Pivot tables are always based on a single table. This means that you have to create a single table of all data you want to report on. Usually this means that you have to create a full join of all the data in your database. If you want to preserve all possible dimension combinations without sales (as we did for only months in previous query), this rapidly leads to a huge table. Even in our very small CustomerSales database, this leads to a table with over 2 million rows: you won't be able to load this in Excel.

Wide vs. long format SQL tables and SQL query results are usually in long format. Suppose we have the following table

```sql
select *
from ( values
    (20, '2017-10-21', 103, 6),
    (22, '2017-10-21', 103, 4),
    (22, '2017-10-21', 101, 2),
    (20, '2017-11-24', 103, 4),
    (20, '2017-11-24', 102, 3)
) as T(custno, orderdate, prodno, qty)
;
```

This yields the following table in long format:

custno | orderdate  | prodno | qty
-------|------------|--------|----
20     | 2017-10-21 | 103    | 6
22     | 2017-10-21 | 103    | 4
22     | 2017-10-21 | 101    | 2
20     | 2017-11-24 | 103    | 4
20     | 2017-11-24 | 102    | 3

Before creating the chart feeder data, it is necessary to think about what and how you want to present in your visual. Suppose I want to present for each customer, for each month a column bar representing the quantity sold if the specific product. I want customers on the X-axis, months on slicers and products as series.

The same table in wide format with products on columns, would yield:

```sql
with cte as (
    select *
    from ( values
        (20, '2017-10-21', 103, 6),
        (22, '2017-10-21', 103, 4),
        (22, '2017-10-21', 101, 2),
        (20, '2017-11-24', 103, 4),
        (20, '2017-11-24', 102, 3)
    ) as T(custno, orderdate, prodno, qty)
)
select custno, datename(m, orderdate) as mnth,
    sum(case when prodno = 101 then qty else 0 end) as onions,
    sum(case when prodno = 102 then qty else 0 end) as beans,
    sum(case when prodno = 103 then qty else 0 end) as potatoes,
    sum(case when prodno = 104 then qty else 0 end) as cabbage
from cte
group by custno, datename(m, orderdate)
;
```

custno | orderdate  | onions | beans | potatoes | cabbage
-------|------------|--------|-------|----------|--------
20     | 2017-10-21 | 0      | 0     | 6        | 0
22     | 2017-10-21 | 2      | 0     | 4        | 0
20     | 2017-10-24 | 0      | 3     | 4        | 0

When table are to be presented to humans, the wide format tends to be better as it is more compact.

Excel wants its chart feeders to be in wide format too. This holds for both regular charts and pivot charts. As pivot charts are based on pivot tables, the chart feeder is almost always in wide format.

Sometimes Excel can express its own will when visualizing data from a chart feeder. The result may not always be to your liking. However, using the Select Data command from the Chart Tools > Design ribbon will always lead to the visual you had in mind.

## 6. Compare revenue figures

**Compare monthly zwezerik revenue in 2014 and 2013.**

We get a straightforward solution by slightly adapting the previous query:

```sql
select year, monthno, monthname, isnull(sum(linetotal), 0) as revenue
from factSales s
join dimProduct p
    on p.prodno = s.prodno
    and proddesc = 'zwezerik'
right join dimDate
    on orderdate = dat
where year in (2014, 2013)
group by year, monthno, monthname
order by year, monthno
;
```

Create a column chart of the result in Excel.

For an Excel chart, it's easier to have the result in pivot format. Different columns are translated into different series in a chart; that is exactly what we want if we want to compare without using a pivot table.

To get different columns, we use a case/when in the select rather than the complicated pivot table-operator.

```sql
select monthno, monthname,
    isnull(sum(case year when 2013 then linetotal else 0 end), 0) as rev13,
    isnull(sum(case year when 2014 then linetotal else 0 end), 0) as rev14
from factSales s
join dimProduct p
    on p.prodno = s.prodno
    and proddesc = 'zwezerik'
right join dimDate
    on orderdate = dat
where year in (2014, 2013)
group by monthno, monthname
order by monthno
;
```

Note that we must no longer group on year if we wish to separate the year in each line of the result set.

## 7. Revenue as percentage of total

**Calculate the monthly revenue of zwezerik in 2014 as a percentage of total revenue of that month.**

Zwezerik's percentual contribution to the monthly revenue is of course zwezerik's part divided by total revenue. Starting from query 5's solution, we could solve it like:

```sql
select monthname,
    isnull(
        sum(linetotal)/(
            select sum(linetotal)
            from factSales
            where year(orderdate) = 2014
            and month(orderdate) = monthno
        ),
        0
    ) as [%rev]
from dimDate
cross join dimProduct p
left join factSales s
    on p.prodno = s.prodno
    and orderdate = dat
where year = 2014
and proddesc = 'zwezerik'
group by monthno, monthname
order by monthno
;
```

Here we used a so-called correlated subquery: the subquery references a column value in the outer query. As a consequence, the subquery should be re-calculated for each row in the outer query, and hence are usually expensive performance wise.

## 8. Revenue as percentage of total - 2

**Calculate the monthly revenue in 2014 of each product category as a percentage of total revenue of that month.**

This is a generalization of previous question, but now for product category instead of product A solution in line with previous solution would be:

```sql
select monthname, catcode,
    isnull(
        sum(linetotal)/(
            select sum(linetotal)
            from factSales
            where year(orderdate) = 2014
            and month(orderdate) = monthno
        ),
        0
    ) as [%rev]
from dimDate
cross join dimProduct p
left join factSales s
    on p.prodno = s.prodno
    and orderdate = dat
where year = 2014
group by monthno, monthname, catcode
order by monthno
;
```

Because of the generalization to all product categories, we can make use of a window function:

```sql
select monthname, catcode,
    isnull(sum(linetotal)/(sum(sum(linetotal)) over (partition by monthno)), 0) as [%rev]
from dimDate
cross join dimProduct p
left join factSales s
    on p.prodno = s.prodno
    and orderdate = dat
where year = 2014
group by monthno, monthname, catcode
order by monthno
;
```

Note that in the sum(sum()) construction, the outer sum() is the windows function that goes with the partition by whereas the inner sum() is the aggregate function that goes with the group by.

In an Excel pivot table this query would be calculated using the % of row total measure with product category on rows. An SQL solution that mimics this pivot table solution would be:

```sql
select monthname,
    isnull(sum(case when catcode = 'bio' then linetotal else 0 end)/sum(linetotal), 0) as bio,
    isnull(sum(case when catcode = 'lux' then linetotal else 0 end)/sum(linetotal), 0) as lux,
    isnull(sum(case when catcode = 'zuv' then linetotal else 0 end)/sum(linetotal), 0) as zuv
from dimDate
cross join dimProduct p
left join factSales s
    on p.prodno = s.prodno
    and orderdate = dat
where year = 2014
group by monthno, monthname
order by monthno
;
```

## 9. Individual vs average sales

**Compare yearly individual sales person revenue with (yearly) average sales person revenue. Show results in a chart.**

The difficulty in this query is that we have to mix addition for a single salesperson with addition (and averaging) for all salespersons. In other words: we have to group on year and name and on year only in the same query. This can only be established using a subquery with its own grouping context:

```sql
select year(orderdate) as year, name as salesrep, sum(linetotal) as revenue, (
    select sum(linetotal)/count(distinct salesrep)
    from factSales si
    where year(si.orderdate) = year(so.orderdate)
) as avgRevenue
from factSales so
join dimSalesPerson
    on salesrep = empno
group by year(orderdate), name
order by year, salesrep
;
```

The inner query is called a correlated subquery because we link the year in the inner query to the year of the current result line in the outer query. Because of this correlation, the inner query must be evaluated for each result line in the outer query.

## 10. Cumulative revenue

**Calculate cumulative monthly revenue per product in 2014. Show results in a chart.**

First use a cross join to generate all month/product combinations. Then use a left (outer) join to preserve month/product combinations without sales:

```sql
select monthno, monthname, proddesc, isnull(sum(linetotal), 0) as revenue
from dimDate
cross join dimProduct p
left join factSales s
    on s.prodno = p.prodno
    and orderdate = dat
where year = 2014
group by monthno, monthname, proddesc
;
```

To calculate the cumulative monthly revenue, instead of the monthly revenue, we use a window function:

```sql
select monthno, monthname, proddesc,
    isnull(sum(sum(linetotal)) over (partition by proddesc order by monthno), 0) as cumrev
from dimDate
cross join dimProduct p
left join factSales s
    on s.prodno = p.prodno
    and orderdate = dat
where year = 2014
group by monthno, monthname, proddesc
;
```

Note the sum(sum()): the outer sum() is the window function, whereas the inner sum() is the aggregate function we used in our first step. The partition by creates a window of all lines having the same proddesc in a month/proddesc group.

## 11. YoY cumulative sales increase

**Calculate monthly year-over-year cumulative sales increase (percentage) for each product category in 2014.**

This is a combination of two previous demo queries:

```sql
with cte as (
    select monthno, monthname, catcode,
        isnull(sum(sum(case year when 2013 then linetotal end)) over (partition by catcode order by monthno), 0) as cumrev13,
        isnull(sum(sum(case year when 2014 then linetotal end)) over (partition by catcode order by monthno), 0) as cumrev14
    from dimDate
    cross join dimProduct p
    left join factSales s
        on s.prodno = p.prodno
        and orderdate = dat
    where year in (2013, 2014)
    group by monthno, monthname, catcode
)
select monthno, monthname, catcode,
    (cumrev14 - cumrev13)/cumrev13 as [YoY Rev Increase %]
from cte
;
```

## 12. Average of last two

**Calculate for each customer the average order value of last two orders.**

First generate a list of order values ordered by order date (most recent on top)

```sql
select custno, orderno, orderdate, sum(linetotal) as ordVal
from factSales
group by custno, orderno, orderdate
;
```

Then add ranking, select top 2 per customer and calculate average:

```sql
with cte as (
    select custno, orderno, orderdate, sum(linetotal) as ordVal,
        row_number() over (partition by custno order by orderdate desc) as rnr
    from factSales
    group by custno, orderno, orderdate
)
select custno, avg(ordval)
from cte
where rnr <= 2
group by custno
;
```

Alternatively with a cross apply (see also this demo for a (rather elaborate) discussion of using either a ranking or a cross apply solution):

```sql
select custno, name, avg(ordVal) as avgOrdVal
from dimCustomer c
cross apply (
    select top (2) sum(linetotal) as ordval
    from factSales s
    where s.custno = c.custno
    group by orderno, orderdate
    order by orderdate desc
) s
group by custno, name
;
```

```sql
select custno, orderdate
from factSales s
where ordno in (
    select top (2) ordno
    from factSales si
    where si.custno = s.custno
)
;
```

## 13. Weekday revenue percentage

**Calculate for each customer the percentage of orders placed on a weekday in 2014.**

```sql
select custno,
    avg(case when datepart(dw, orderdate) between 2 and 6 then 1.0 else 0.0 end) as wkdy
from factSales
where year(orderdate) = 2014
group by custno
;
```

## 14. Ranking revenue figures

**Give a weekday ranking for February 2014 with the best selling (revenue) weekday on top.**

```sql
select datename(dw, orderdate), sum(linetotal) as rev,
    rank() over(order by sum(linetotal) desc) as rnk
from factSales
where orderdate between '2014-02-01' and '2014-02-28'
group by datename(dw, orderdate)
;
```

Looks ok, but we miss Saturday: there have been no sales on Saturday in Feb 2014. Clearly it's important to also present the days without sales. Let's fix it by outer joining with dimDate:

```sql
select dayName, isnull(sum(linetotal), 0) as rev,
    rank() over(order by sum(linetotal) desc) as rnk
from dimDate
left join factSales
    on orderdate = dat
where dat between '2014-02-01' and '2014-02-28'
group by dayName, dayInWeek
order by dayInWeek
;
```

Note that the where clause should involve the date from dimDate rather than from factSales. If not, the days without sales would have had an NULL orderDate and the line would have left out by the filter leading again to a result without days with no sales.

Give a weekday ranking for February 2014 with the best selling (# transactions) weekday on top.

You might think: replace the sum with a count and we're all set:

```sql
-- wrong solution
select datename(dw, dat), count(linetotal) as [#tx],
    rank() over(order by count(linetotal) desc) as rnk
from dimDate
left join factSales
    on orderdate = dat
where dat between '2014-02-01' and '2014-02-28'
group by datename(dw, dat), dayInWeek
order by dayInWeek
;
```

Alas, it turns out not to be that simple. factSales has order line granularity meaning that an order can have more than one line. We are only interested in the unique sales transactions:

```sql
select datename(dw, dat), count(distinct orderno) as [#tx],
    rank() over(order by count(distinct orderno) desc) as rnk
from dimDate
left join factSales
    on orderdate = dat
where dat between '2014-02-01' and '2014-02-28'
group by datename(dw, dat), dayInWeek
order by dayInWeek
;
```

Even if we get the same result, the former solution is wrong. We get the same results by coincidence as there where no sales transactions with more than one row in February 2014. Change the filter to February 2013 and you see immediately that the results differ! So remember, even if you test your results, you can still have a wrong query. Testing more thoroughly (using more test sets) and above all remain being critical about your products (and those of others) is the best way to prevent mistakes and avoid ill decisions based on wrongly obtained information.

## 15. ABC labeling

**Calculate ABC labeling for all customers.**

Your top customers belong to the top 20% regarding revenue created. Your inactive customers belong to the tail 30% regarding revenue created. Attach an ABC label to all customers: label top customers A, inactive customers C and the remaining customers B.

Instead of plain ranking with rank(), we now use the percent_rank() window function. As with plain ranking, ordering is key to obtain the ranking percentages you want.

```sql
select custno,
    case
        when percent_rank() over (order by sum(linetotal)) >= 0.8 then 'A'
        when percent_rank() over (order by sum(linetotal)) < 0.3 then 'C'
        else 'B'
    end as label
from factSales
group by custno
order by custno
;
```

Repeating the over () clause looks a bit clumsy, but it's just how the case operator works.

The percent_rank() function is a bit difficult to grasp at first. The lowest ranking group is always 0%. The highest ranking group is always m/(n-1) % where n is the number of rows (observations) and m is the number of rows with a lower value for the column you want to rank on. (Hence, we always start with 0% as no rows will have a lower value than the lowest value.)

Try the query below and change some data in order to get a good understanding of the percent_rank() function. Do not just change and run, but first predict what the result should be. If the result is conform your expectation, you've probably understood. If not, your mental model is wrong and you have to do some more thinking and experimenting.

```sql
select id, percent_rank() over (order by val) [%rank]
from ( values
    (1, 1),
    (2, 1),
    (3, 2),
    (4, 3),
    (5, 3)
) T(id, val)
order by id
;
```

## 16. Basic statistical measures

**What is the number, max, min, mean and standard deviation of the sales value per order?**

```sql
with cte as (
    select sum(linetotal) ordval
    from factSales
    group by orderno
)
select
    count(*) as n,
    min(ordval) as min,
    max(ordval) as max,
    avg(ordval) as mean,
    stdev(ordval) as stdev
from cte
;
```

## 16b. Create a chart feeder for a histogram of sales values per order in Excel.

Best to create a simple chart feeder and let Excel do the the heavy lifting of histogramming as binning is bit of a nuisance in SQL.

```sql
select sum(linetotal) ordval
from factSales
group by orderno
;
```

## 17. Revenue contribution

**Calculate the contribution to revenue (as percentage) of each product category in each region. Present as pivot with product categories on columns and region on rows.**

```sql
select regioncode,
    sum(case when catcode = 'bio' then linetotal end) as bio,
    sum(case when catcode = 'lux' then linetotal end) as lux,
    sum(case when catcode = 'zuv' then linetotal end) as zuv
from dimCustomer c
cross join dimProduct p
left join factSales s
    on s.custno = c.custno
    and s.prodno = p.prodno
group by regioncode
;
```

Calculate the relative contribution to revenue (as percentage) of each product category in each region.

Relative to what? To total regional revenue? To total product category revenue or to total overall revenue? In Excel pivot table terminology: as a % of Row Total, a % of Column Total or a % of Grand Total?

We solve all three:

```sql
-- percentage of regional revenue (% of row total)
select regioncode,
    sum(case when catcode = 'bio' then linetotal end)/sum(linetotal) as bio,
    sum(case when catcode = 'lux' then linetotal end)/sum(linetotal) as lux,
    sum(case when catcode = 'zuv' then linetotal end)/sum(linetotal) as zuv
from dimCustomer c
cross join dimProduct p
left join factSales s
    on s.custno = c.custno
    and s.prodno = p.prodno
group by regioncode
;
```

```sql
-- percentage of product category revenue (% of column total)
select regioncode,
    sum(case when catcode = 'bio' then linetotal end)/(
        select sum(linetotal)
        from dimProduct p
        join factSales s
            on s.prodno = p.prodno
        where catcode = 'bio'
    ) as bio,
    sum(case when catcode = 'lux' then linetotal end)/(
        select sum(linetotal)
        from dimProduct p
        join factSales s
            on s.prodno = p.prodno
        where catcode = 'lux'
    ) as lux,
    sum(case when catcode = 'zuv' then linetotal end)/(
        select sum(linetotal)
        from dimProduct p
        join factSales s
            on s.prodno = p.prodno
        where catcode = 'zuv'
    ) as bio
from dimCustomer c
cross join dimProduct p
left join factSales s
    on s.custno = c.custno
    and s.prodno = p.prodno
group by regioncode
;
```

We can make this a bit less verbose with a common table expression:

```sql
-- percentage of product category revenue (% of column total)
with cte as (
select regioncode, catcode, sum(linetotal) as total
from dimCustomer c
cross join dimProduct p
left join factSales s
    on s.custno = c.custno
    and s.prodno = p.prodno
group by regioncode, catcode
)
select regioncode,
    sum(case when catcode = 'bio' then total end)/(select sum(total) from cte where catcode = 'bio') as bio,
    sum(case when catcode = 'lux' then total end)/(select sum(total) from cte where catcode = 'lux') as lux,
    sum(case when catcode = 'zuv' then total end)/(select sum(total) from cte where catcode = 'zuv') as zuv
from cte co
group by regioncode
;
```

Finally the query to calculate the percentage of total revenue for each region/product category combination:

```sql
-- percentage of overall revenue (% of grand total)
select regioncode,
    sum(case when catcode = 'bio' then linetotal end)/(sum(sum(linetotal)) over()) as bio,
    sum(case when catcode = 'lux' then linetotal end)/(sum(sum(linetotal)) over()) as lux,
    sum(case when catcode = 'zuv' then linetotal end)/(sum(sum(linetotal)) over()) as zuv
from dimCustomer c
cross join dimProduct p
left join factSales s
    on s.custno = c.custno
    and s.prodno = p.prodno
group by regioncode
;
```

We do not necessarily need a window function as denominator as a straightforward subquery will do as well:

```sql
...
    sum(case when catcode = 'bio' then linetotal end)/(select sum(linetotal) from factSales) as bio,
...
```
