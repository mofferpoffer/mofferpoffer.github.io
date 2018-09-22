# Exercise Queries

Always validate your results by checking it against a joinThemAll table in Excel or a simple SQL check query!

If you want to get inspiration to solve either of these exercise queries, here's a list of demo queries you could study to solve the exercise queries:

Demo Queries | Exercise Queries
-------------|-----------------
1            | 1, 2
2            | 1, 2
3            | 4
4            | 9
5            | 12
6            | 6, 8, 11
7            | 12
8            | 14
9            | 11
10           |
11           |
12           |
13           |
14           |
15           |

### 1. Average order value

**What is the average order value per city in 2014?**

This is a warming up exercise.

Hint: factSales has order line granularity. Do not calculate the average order line value!

```sql
select city, sum(linetotal)/count(distinct orderno) as [avg order value]
from dimCustomer c
join factSales s
    on s.custno = c.custno
where year(orderdate) = 2014
group by city
;
```

### 2. Average number of items sold (1)

**What is for each product for each year the average number of items sold in an order?**

```sql
select proddesc, year(orderdate) as year,
    sum(quantity)/count(distinct orderno) as [avg # items sold]
from dimProduct p
join factSales s
    on s.prodno = p.prodno
group by proddesc, year(orderdate)
;
```

### 3. Average number of items sold (2)

**What is for each product for each month in 2014 the average number of items sold in an order?**

Order by month name, then by product description.

Hint: you don't want alphabetic month name ordering. To get proper Jan - Dec ordering, you need to use the month number.

```sql
select monthname as mnth, proddesc,
    sum(quantity)/count(distinct orderno) as [avg # items sold]
from dimDate
cross join dimProduct p
left join factSales s
    on s.prodno = p.prodno
where year(orderdate) = 2014
group by monthno, monthname, proddesc
order by monthno, proddesc
;
```

### 4. Proportion of yearly revenue

**Calculate for each customer for each month in 2014 the proportion of yearly revenue.**

Your answer should look something like:

name     | monthname | monthly proportion
---------|-----------|-------------------
Williams | Jan       | 12.30 %
Williams | Feb       | 27.21 %
Williams | Mar       | 0.00 %
...      | ...       | ...

Hint: in each line divide the monthly revenue by the yearly revenue for that specific customer.

```sql
select name, monthname,
     format(
         isnull(
            sum(linetotal)/sum(sum(linetotal)) over (partition by name),
            0
        ), 'p'
    ) as [monthly proportion]
from dimDate
cross join dimCustomer c
left join factSales s
    on orderdate = dat
    and s.custno = c.custno
where year = 2014
group by name, monthno, monthname
order by name, monthno
;
```

### 5. Year-over-year sales

**Calculate monthly year-over-year sales increase (percentage, non cumulative) for each city for all years.**

Hint: look at the YoY demo query and take into account that we don't want cumulative revenue.

```sql
with cte as (
    select monthno, monthname, city,
        sum(case year when 2014 then linetotal end) as rev,
        sum(case year when 2013 then linetotal end) as revLY
    from dimDate
    cross join dimCustomer c
    left join factSales s
        on s.custno = c.custno
        and orderdate = dat
    where year in (2013, 2014)
    group by monthno, monthname, city
)
select city, monthno, monthname, city, rev, revLY,
    (rev - revLY)/revLY as [YoY Rev Increase %]
from cte
;
```

Alternatively with the lag() window function

```sql
with cte as (
    select year, monthno, monthname, city, isnull(sum(linetotal), 0) as rev,
        lag(sum(linetotal)) over (partition by city, monthno order by year) as revLY
    from dimDate
    cross join dimCustomer c
    left join factSales s
        on s.custno = c.custno
        and orderdate = dat
    where year in (2013, 2014)
    group by monthno, year, monthname, city
)
select city, month, (rev - revLY)/revLY as [% YoY rev increase]
from cte
where year = 2014
;
```

Advantage of the latter solution is that you can easily generalize it to all years for which you have data: just remove the where clauses.

### 6. Days between orders

**What is for each customer the number of days between their last and last but one order?**

Hints:

1. use the row_number() window function to rank orders based on their order date
1. then select the most recent 2 (the most recent having the highest rank)

First we solve a more general question: what are last and last but one order dates for each customer?

We use the row_number() window function to tank each order line. Since factSales has order line granularity, you have to group on orderno. A tie breaker for order on the same date (e.g. orderno) is not necessary

```sql
with cte as (
    select custno, orderdate,
    row_number() over (partition by custno order by orderdate desc) as rno
    from factSales
    group by orderno, custno, orderdate -- ! watch this: possibly more than 1 orderline per orderno!
)
select custno, orderdate
from cte
where rno <= 2
;
```

Now it's easy to specialize to calculating the difference between both dates

```sql
with cte as (
    select custno, orderdate,
        row_number() over (partition by custno order by orderdate desc) as rno
    from factSales
    group by orderno, custno, orderdate
)
select custno, datediff(d, min(orderdate), max(orderdate)) as diff
from cte
where rno <= 2
group by custno
;
```


A slightly different type of solution is to use a lead() in combination with row_number(). The lead() is used to calculate for each row (order) the difference in days with the immediately preceding order.

```sql
with cte as (
    select custno, orderdate,
        row_number() over (partition by custno order by orderdate desc) as rno,
        datediff(d, lead(orderdate) over (partition by custno order by orderdate desc), orderdate) as diff
    from factSales
    group by orderno, custno, orderdate
)
select custno, diff
from cte
where rno = 1
;
```

Note: though a bit more difficult to grasp, we can also solve this using the cross apply table operator:

```sql
select c.custno, datediff(d, min(orderdate), max(orderdate)) as diff
from dimCustomer c
cross apply (
    select top (2) custno, orderdate
    from factSales s
    where s.custno = c.custno
    group by orderno, custno, orderdate  -- factSales has order line granularity!
    order by orderdate desc, orderno
) t
group by c.custno
;
```

Finally, a more traditional (but properly working) approach would be to use a correlated subquery. I suspect this query to be less performant than the cross apply solution as in this solution the factSales will be re-queried for each line in factSales, whereas in the cross apply solution it will be queried for each customer

```sql
select custno, datediff(d, min(orderdate), max(orderdate)) as diff
from factSales s
where orderno in (
    select top (2) orderno
    from factSales si
    where si.custno = s.custno
    group by orderno, orderdate
    order by orderdate desc
)
group by custno
;
```

### 7. Average time between orders

**What is for each customer the average time lag between 2 orders of the 5 most recent orders of this customer?**

So you have to collect the 5 most recent orders per customer, calculate the 4 time lags between subsequent orders and then calculate the average of these 4 time lags.

Hints:

1. use the lead() window function to look something up in a successor row
1. use the avg() function to calculate an average
1. check/test your results

```sql
with cte as (
select custno, orderdate,
    row_number() over (partition by custno order by orderdate desc, orderno) as rno,
    datediff(d, lead(orderdate) over (partition by custno order by orderdate desc), orderdate ) as tlag
from factSales
group by orderno, custno, orderdate
)
select custno,  avg(convert(float, tlag)) as [avg time lag]
from cte
where rno <= 4
group by custno
;
```

Note that we calculate the following average expression: (d2 - d1) + (d3 - d2) + (d4 - d3) + (d5 - d4)/4 which can be simplified to (d5-d1)/4. In SQL:

```sql
with cte as (
    select custno, orderdate,
        row_number() over (partition by custno order by orderdate desc) as rno
    from factSales
    group by orderno, custno, orderdate
)
select custno, datediff(d, min(orderdate), max(orderdate)) as diff
from cte
where rno in (1, 5)
group by custno
;
```

The main difference with the first solution is that if you don't have at least 5 dates for different orders, the latter gives 0 (min and max are the same) whereas the first solution gives the average of all dates it found, even if there were less than 5. You can, however, adapt the second solution to get the same behavior:

```sql
with cte as (
    select custno, orderdate,
        row_number() over (partition by custno order by orderdate desc) as rno
    from factSales
    group by orderno, custno, orderdate
)
select custno,
    convert(float, datediff(d, min(orderdate), max(orderdate)))/(count(*) - 1) as diff
from cte
where rno <= 5
group by custno
;
```

### 8. Cumulative revenue

**Calculate total revenue and total cumulative revenue for each year, for each month**

```sql
select year, monthname, sum(linetotal) as rev,
    sum(sum(linetotal)) over (order by monthno) as cumrev
from dimDate d
join factSales s
    on orderdate = dat
group by year, monthname, monthno
;
```

**Visualize the result in a combo chart with years on slicers**

### 9. Cumulative revenue pivoted

Calculate cumulative revenue for each year, each month, each regioncode and each product category. Present in pivot format with product category on columns

```sql
select year, monthno, monthname, regioncode,
    sum(sum(case when catcode = 'bio' then linetotal end)) over (partition by year, regioncode order by monthno) as Bio,
    sum(sum(case when catcode = 'lux' then linetotal end)) over (partition by year, regioncode order by monthno) as Lux,
    sum(sum(case when catcode = 'zuv' then linetotal end)) over (partition by year, regioncode order by monthno) as Zuv
from dimDate
cross join dimCustomer c
cross join dimProduct p
left join factSales s
    on orderdate = dat
    and s.custno = c.custno
    and s.prodno = p.prodno
group by year, monthno, monthname, regioncode
;
```

**Visualize in a stacked bar chart (product categories as series), with months on x-axis and years and region code on slicers.**

### 10. Percentage of total

**Calculate the monthly revenue in 2014 of each product category as a percentage of total revenue for that product category in that year.**

Hint: the demo query calculates an Excel equivalent of percentage of row total. Here you have to calculate the Excel percentage of column total equivalent.

```sql
select monthname, catcode,
    sum(linetotal)/(sum(sum(linetotal)) over (partition by catcode))
from dimDate
cross join dimProduct p
join factSales s
    on s.prodno = p.prodno
    and orderdate = dat
where year(orderdate) = 2014
group by monthno, monthname, catcode
;
```

If you want to present in pivot format with product categories on columns, it is easiest to do so with previous query as a common table expression:

```sql
with cte as (
select monthno, monthname, catcode,
    sum(linetotal)/(sum(sum(linetotal)) over (partition by catcode)) as [%rev]
from dimDate
cross join dimProduct p
join factSales s
    on s.prodno = p.prodno
    and orderdate = dat
where year(orderdate) = 2014
group by monthno, monthname, catcode
)
select monthname,
    sum(case when catcode = 'bio' then [%rev] end) as bio,
    sum(case when catcode = 'lux' then [%rev] end) as lux,
    sum(case when catcode = 'zuv' then [%rev] end) as zuv
from cte
group by monthno, monthname, monthname
order by monthno
;
```

Although summing percentages from the cte may look ridiculous, it is save here, because you will only sum a single value for a particular month/product category combination. So you might just as well have used a max() or a min() in the query above.

### 11. Best spending customers

**Which customers in each region spent most in 2014?**

```sql
with cte as (
    select regioncode, custno, name, sum(linetotal) as totalSpent,
        row_number() over (partition by regioncode order by sum(linetotal) desc) as rno
    from dimCustomer c
    join factSales s
        on s.custno  = c.custno
    group by regioncode, c.custno, name
)
select regioncode, c.custno, name, totalSpent
from cte
where rno = 1
;
```

### 12. Periodical growth

**What is for each product category the yearly growth (percentage) of quantity sold in 2013 and 2014?**

```sql
with cte as (
    select year(orderdate) as yr, catcode, sum(linetotal) as totSales,
    lag(sum(linetotal)) over (partition by catcode order by year(orderdate)) as totSalesLY
    from dimProduct p
    join factSales s
        on s.prodno = p.prodno
    group by year(orderdate), catcode
)
select yr, catcode, (totSales - totSalesLY)/totSalesLY
from cte
where totSalesLY is not null
;
```

### 13. Weekend and weekday buyers

Weekend buyers place at least 40% of their orders in weekends.

**Which customers can be labeled as weekend buyers?**

Hints:

1. look at query 11 from the demo series
1. use a having clause if you have a condition on a group result
1. keep in mind that factSales has order line granularity

```sql
with cte as (
    -- select all unique orders with their customer and order date
    select distinct custno, orderno, orderdate
    from factSales
)
select custno
from cte
group by custno
having avg(case when datepart(dw, orderdate) in (1, 7) then 1.0 else 0.0 end) >= 0.4
;
```

Well, that's a funny looking having clause ... It is a pattern you can often use when you have to produce a percentage of countable events, in this case order transactions in the weekend as a percentage of all order transactions. In the case we label each positive transaction with a 1 and each negative transaction with a 0. Then we calculate the average of this column of 1-2 and 0-s. That average is exactly the percentage of 1-s or the percentage of positive events.

### 14. Maximum lag time

**What is for each customer the longest period (in days) between two consecutive orders in January 2014?**

Note that you can only calculate this for customers that placed at least 2 orders in 2014.

```sql
with cte as (
    select custno,
        datediff(d, lead(orderdate) over (partition by custno order by orderdate desc), orderdate) as diff
    from factSales
    where year(orderdate) = 2014 and month(orderdate) = 1
    group by custno, orderno, orderdate
)
select custno, max(diff) as maxdiff
from cte
where diff is not null
group by custno
;
```

### 15. Orders per day of week

**Present for each customer their number of orders per day of week in 2014 in an Excel column chart**

Hint: look at queries 11 and 12.

First create a result without pivoting:

```sql
select custno, datename(dw, orderdate) as dy, count(distinct orderno) as [#orders]
from factSales
where year(orderdate) = 2014
group by custno, datename(dw, orderdate)
;
```

Then pivot. First we pivot using the case operator:

```sql
select custno,
    sum(case datepart(dw, orderdate) when 1 then 1 else 0 end) as Sunday,
    sum(case datepart(dw, orderdate) when 2 then 1 else 0 end) as Monday,
    sum(case datepart(dw, orderdate) when 3 then 1 else 0 end) as Tuesday,
    sum(case datepart(dw, orderdate) when 4 then 1 else 0 end) as Wednesday,
    sum(case datepart(dw, orderdate) when 5 then 1 else 0 end) as Thursday,
    sum(case datepart(dw, orderdate) when 6 then 1 else 0 end) as Friday,
    sum(case datepart(dw, orderdate) when 7 then 1 else 0 end) as Saturday
from factSales
where year(orderdate) = 2014
group by custno
;
```

Looks ok, right? It isn't, however. In the pivoted solution we are counting order lines instead of unique orders. We've lost the distinct feature from the first non pivoted solution. We solve by first filtering the unique custno, orderno, orderdate combination in 2014 in the factSales table:

```sql
with cte as (
    select distinct custno, orderno, orderdate
    from factSales
    where year(orderdate) = 2014
)
select custno,
    sum(case datepart(dw, orderdate) when 1 then 1 else 0 end) as Sunday,
    sum(case datepart(dw, orderdate) when 2 then 1 else 0 end) as Monday,
    sum(case datepart(dw, orderdate) when 3 then 1 else 0 end) as Tuesday,
    sum(case datepart(dw, orderdate) when 4 then 1 else 0 end) as Wednesday,
    sum(case datepart(dw, orderdate) when 5 then 1 else 0 end) as Thursday,
    sum(case datepart(dw, orderdate) when 6 then 1 else 0 end) as Friday,
    sum(case datepart(dw, orderdate) when 7 then 1 else 0 end) as Saturday
from cte
group by custno
;
```

If we join with dimDate, we can make use of the pre-calculated days:

```sql
with cte as (
    select distinct custno, orderno, dayInWeek
    from dimDate
    join factSales
        on orderdate = dat
    where year(orderdate) = 2014
)
select custno,
    sum(case dayInWeek when 1 then 1 else 0 end) as Sunday,
    sum(case dayInWeek when 2 then 1 else 0 end) as Monday,
    sum(case dayInWeek when 3 then 1 else 0 end) as Tuesday,
    sum(case dayInWeek when 4 then 1 else 0 end) as Wednesday,
    sum(case dayInWeek when 5 then 1 else 0 end) as Thursday,
    sum(case dayInWeek when 6 then 1 else 0 end) as Friday,
    sum(case dayInWeek when 7 then 1 else 0 end) as Saturday
from cte
group by custno
;
```

There is no need to left join here, as the explicit weekday column calculation already takes care of that.

Alternatively we can use T-SQL's pivot table operator:

```sql
select custno, Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday
from (
    select custno, datename(dw, orderdate) as dy, count(distinct orderno) as [#orders]
    from factSales
    where year(orderdate) = 2014
    group by custno, datename(dw, orderdate)
) as T1
pivot (
    sum([#orders])
    for dy in (Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday)
) asT2
;
```

Note that Excel will handle the NULLs properly.


### 16. Transactions vs revenue

**Create a (regular) chart in Excel comparing monthly sales transactions with monthly revenue in 2014.**

```sql
select monthno, monthname,
    count(distinct orderno) as [#tx],
    sum(linetotal) as rev
from dimDate
left join factSales
    on orderdate = dat
where year = 2014
group by monthno, monthname
order by monthno
;
```

### 17. Transactions per day of week

**Present for each customer their number of orders per day of week in January 2014.**

Being lazy and copying a proven solution for a new problem is often a good thing. But this solution needs some more.

```sql
-- inferior solution
select custno,
    sum(case datepart(dw, orderdate) when 1 then 1 else 0 end) as Sunday,
    sum(case datepart(dw, orderdate) when 2 then 1 else 0 end) as Monday,
    sum(case datepart(dw, orderdate) when 3 then 1 else 0 end) as Tuesday,
    sum(case datepart(dw, orderdate) when 4 then 1 else 0 end) as Wednesday,
    sum(case datepart(dw, orderdate) when 5 then 1 else 0 end) as Thursday,
    sum(case datepart(dw, orderdate) when 6 then 1 else 0 end) as Friday,
    sum(case datepart(dw, orderdate) when 7 then 1 else 0 end) as Saturday
from factSales
where orderdate between '2014-01-01' and '2014-01-31'
group by custno
;
```

Oops, we lost track of a number of customers now the time frame is a lot smaller. Let's fix that:

```sql
select c.custno,
    sum(case datepart(dw, orderdate) when 1 then 1 else 0 end) as Sunday,
    sum(case datepart(dw, orderdate) when 2 then 1 else 0 end) as Monday,
    sum(case datepart(dw, orderdate) when 3 then 1 else 0 end) as Tuesday,
    sum(case datepart(dw, orderdate) when 4 then 1 else 0 end) as Wednesday,
    sum(case datepart(dw, orderdate) when 5 then 1 else 0 end) as Thursday,
    sum(case datepart(dw, orderdate) when 6 then 1 else 0 end) as Friday,
    sum(case datepart(dw, orderdate) when 7 then 1 else 0 end) as Saturday
from dimCustomer c
left join factSales s
    on s.custno = c.custno
and orderdate between '2014-01-01' and '2014-01-31'
group by c.custno
;
```

Note that the order date condition should not be in the where clause but in the on clause in order to really get all customers. There is no need to outer join dimDate as well, because we explicitly add columns for each weekday. In the result you can see that there haven't been any sales on Mondays, Tuesdays and Thursdays.

### 18. Weekday ranking

**Give for each month in 2014 a weekday ranking with the weekday with the most transactions on top.**

Present in pivot format with weekdays on rows and month names on columns. Show results in an Excel chart.

Hint: look at query 13 from the demo series.

```sql
select datepart(dw, dat) as dyno, datename(dw, dat) dy,
    rank() over(order by count(distinct case monthno when 1 then orderno end) desc) as Jan,
    rank() over(order by count(distinct case monthno when 2 then orderno end) desc) as Feb,
    rank() over(order by count(distinct case monthno when 3 then orderno end) desc) as Mar,
    rank() over(order by count(distinct case monthno when 4 then orderno end) desc) as Apr,
    rank() over(order by count(distinct case monthno when 5 then orderno end) desc) as May,
    rank() over(order by count(distinct case monthno when 6 then orderno end) desc) as Jun,
    rank() over(order by count(distinct case monthno when 7 then orderno end) desc) as Jul,
    rank() over(order by count(distinct case monthno when 8 then orderno end) desc) as Aug,
    rank() over(order by count(distinct case monthno when 9 then orderno end) desc) as Sep,
    rank() over(order by count(distinct case monthno when 10 then orderno end) desc) as Oct,
    rank() over(order by count(distinct case monthno when 11 then orderno end) desc) as Nov,
    rank() over(order by count(distinct case monthno when 12 then orderno end) desc) as Dec
from dimDate
left join factSales
    on orderdate = dat
where year = 2014
group by datename(dw, dat), datepart(dw, dat)
order by datepart(dw, dat)
;
```

Tsss, that's an odd looking solution. And not really intuitive either. And yet the only thing we did was substitute the count(case ...) for the count(orderdate) in previous solution.

If I had to solve this query without having solved the previous one first, I don't think I would have succeeded. That's why it is a very practical and good strategy to always try to solve a simpler version of your query to get some sense of structure and solution direction.

It is debatable whether this result set format is the most convenient to base visuals on in charts. Perhaps the unpivoted version below is more suited in Excel:

```sql
select monthno, monthname, datepart(dw, dat) as dyno, datename(dw, dat) dy,
    rank() over(partition by monthno order by count(distinct orderno) desc) as rnk
from dimDate
left join factSales
    on orderdate = dat
where year = 2014
group by monthno, monthname, datename(dw, dat), datepart(dw, dat)
order by monthno, datepart(dw, dat)
;
```

Because dimDate has additional columns for month number and month name, we can use them in this query. However, there are no additional columns for weekday and weekday number, so we have to calculate them.

### 19. ABC labeling

**Calculate ABC labeling for all product categories.**

Hint: look at query 15 from the demo series.

```sql
select catcode,
    case
        when percent_rank() over (order by sum(linetotal) desc) <= 0.2 then 'A'
        when percent_rank() over (order by sum(linetotal) desc) >= 0.7 then 'C'
        else 'B'
    end as label
from dimProduct p
join factSales s
    on p.prodno = s.prodno
group by catcode
order by catcode
;
```
