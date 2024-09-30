---
layout: post
title: Optimizers: The Good, the bad, and the "Does My System do This?"
gh-repo: duckdb/duckdb-web
gh-badge: [star, fork, follow]
tags: [Databases, DuckDB, Optimizers]
comments: true
---


An optimizer can improve query performance by 100x. This is not the case with 99% percent of queries, but what if it is the one query you run for every dashboard refresh? Would you still choose to run your handwritten query without optimizations? Query optimizers are so advanced now they can perform optimizations that humans cannot handwrite.

So why is it important to have an optimizer for data analytics? Well, there are a few reasons. Sometimes the general physical makeup of the data can influence the performance of a non-optimized query. Other times, changes in the make up of the data can render a previous implementation of handwritten ETL code innefficient. And finally, some logical optimizations are impossible to write by hand.

## The problem to optimize

We will use the tpch dataset for this problem. To compare the performance with optimization and without optimization, each scenario will be performed with the optimizer on and with the optimizer off. The queries will be written specifically to show the important of one optimizer. (yes, there are multiple optimizers).


### Optimizing based on operators


Some optimizations are purely optimizations based on how the data physically moves through the CPU&memory to execute the logical task. Some of these optimizations are possible by hand, but usually they are not. Below is a query that takes advantage of 3 optimizations that occur because the optimize how data physically moves from memory to the CPU.


```sql
select c_custkey, count(*) count 
from customer, orders 
where 
	c_custkey = o_custkey and 
	c_nationkey = 20 
group by c_custkey 
order by count desc 
limit 5;
```

This will show us the top 5 customers with the highest number of orders from nation with nation key 20. 

If we run this query with and without optimization, these are the results

| With Optimization    | 0.016 |
|----------------------|-------|
| Without Optimization | >1min |


The problem with no optimization is that the filter is applied after a cross product. An optimization that most systems would have is to make the join an inner join and have the filter on just the table. We can hand-rewrite the query as follows.


```sql
select c_custkey, count(*) count 
from 
	(select * from customer where c_nationkey = 20) customer
JOIN orders ON 
	c_custkey = o_custkey
group by c_custkey 
order by count desc 
limit 5;
```

Now the execution time looks like the following
| With Optimization    | 0.016 |
|----------------------|-------|
| Hand-Optimized       | 0.060 |
|----------------------|-------|
| Without Optimization | >1min |


But this is a hand optimized query. Look at how much uglier it is! Imagine doing this for every query you would need to write. And even then, the machine optimized query is 6x faster!

Let us now investigate where the performance is going.

<!-- Build side probe side optimizer -->
<!-- filter pushdown into scans -->
<!-- TopN optimizer -->

Suppose we have a simple join query in our ETL pipeline. We want to join orders and customers so we can have the customer information for every order and v.v. If we are working with a system that does not have a join order optimizer this can have detrimental effects.

From a pure brain to SQL perspective, the query could look like this 
```sql
select c_custkey, count(*) from customer, orders where c_custkey = o_custkey;
```

But without any optimizations, this gives you a cross product. So let's hand optimize the query a little bit to give ourselves a fair fight with the optimizer. 
```sql
select * from customer join  orders on c_custkey = o_custkey;
```

the above query is a bit better. On my machine it completes in `2.694` seconds on scale factor 10.

Now, let's run the same query with the optimizer on 


```sql
pragma enable_optimizer;
select * from customer join  orders on c_custkey = o_custkey;
```

Wow! That took `1.132` seconds, so it more than ~2x faster. Why?

Well, Join Order Optimization came into play. This optimizer knows what physical operators will planned, and can then optimize for those specific operators. In this case, a physical hash join will be planned, without boring you with details, it is better to have the smaller table on the right hand side of the join (although this varies between systems). The join order optimizer will inspect the table cardinalities and perform this swap.


You may now think, oh, but I can just do that myself when I write the query. You may be right, but that leads me to my next reason why optimizers are extremely important

### Optimizing the non-human-optimizable -- Or when data changes

<!-- join order optimizer -->
<!-- Top N optimizer -->
<!-- statistics propagation -->
<!-- unused columns etc. -->
<!-- join order optimizer -->

Take join order optimization, what happens when your data changes? What happens




### Ok, I need optimization, what systems have it?

Most databases have optimization. Systems that don't have optimizers R (data.table, dplyr, ...). Python libraries like pandas and datafusion??


Well, in many cases an ETL pipeline, or a SQL query, or some data transfer is usually a set it and forget it type of thing. This means that once the code to perform some logic is written, it is no longer actively maintained. The problem, however, is that while the code is no longer maintained, the physical make up of the data it is accessing does change. 




What can a good optimizer do that a normal human being might miss?
- As data changes in an ETL pipeline, the most efficient way to retreive the data can also change. The make-up of the data can start changing to the point where cheap data retreivals become much more expensive because more records were added. So now, instead of this retreival being one of your first operations it should be your last.
- Think about Regex-range optimizations (I still don't know what these are)
     - But you do have regex to like optimizations.
- Expression re-writing. So many here, Constant folding, distributivity rule, aritmetic simplification rule, case simplification rule, conjunction simplification etc.
- **Unused columns** & **column lifetime analyzer**. This is a big one if you write your queries incorrectly. You end up holding a lot of data in memory that goes completely unused.
- Common aggregates. Less important, but if you want to check percentages of count(i)/count(*), you could end up double counting certain aggregate expressions.
- TopN optimizer, if you do an order by and a limit, then you don't need to order all values, just the top N values that you've seen. This agains helps with memory costs.
- Reorder Filter. What filters to apply first based on cost, but really this should be based on statistics.
- Filter pull up and filter pushdown
- In clause rewriter.


 
- Think about filter pushdown in the taxi case
  https://github.com/Tmonster/duckplyr_demo/blob/main/duckplyr/q03_popular_manhattan_cab_rides.R ? The last filter is on pickup dropoff = manhattan, but then the boroughs can also be filtered to only project manhattan stuff.
```
select pickup.zone pickup_zone, dropoff.zone dropoff_zone, count(*) as num_trips
from 
	zone_lookups pickup, 
	zone_lookups dropoff,
	taxi_data_2019 data
where 
	pickup.LocationID = data.pickup_location_id and
	dropoff.LocationID = data.dropoff_location_id
	and
	pickup.Borough = 'Manhattan' and 
	dropoff.Borough = 'Manhattan'
group by pickup_zone, dropoff_zone
order by num_trips desc;
```

Naively, someone might not filter the zone_lookups to only include only the Manhattan borough.

- And potentially modifying certain ETL parths so that a query selects extra data (suppose a window expression). How can a naive fix in a optimizerless system compare to a naive fix in a regular system.



I want to know,
Why is it good to have an optimizer?
What systems have optimizer?
What systems allow you to turn on/off certain parts of the optimizer?
Can you see how your optimizer is affecting the plan?

1. [Polars](https://docs.pola.rs/user-guide/lazy/optimizations/)
   - You can see the query plan by calling show_graph https://docs.pola.rs/user-guide/lazy/query-plan/
   - 
3. [Datafusion](https://pypi.org/project/datafusion/)
4. Databend
   - ???
6. [Spark](https://www.databricks.com/glossary/catalyst-optimizer) catalyst optimizer.
  - apparently you can see the optimizer effects by looking at the string() method. https://stackoverflow.com/questions/55614122/how-do-i-get-a-spark-dataframe-to-print-its-explain-plan-to-a-string
  - not much else about what optimizers are available on the system.
5. [Pandas](https://pandas.pydata.org/docs/user_guide/enhancingperf.html)
  - I don't think there really is any optimization workflow happening here.


WHy is maintaining an optimizer difficult?
So many downstream effects.




Why is working on an optimizer so difficult? I guess normally the workflow for performance programming is "Here is a problem with some variables, it needs to be faster", and then you can work on it until it gets faster. This is the case with optimization, except once the thing is faster, you have to make sure **nothing else is slower**, and this is the part that takes forever. I implemented the Join Order Optimizer for DuckDB, then refactored it, then refactored it again. Most of the time spent on these PRs was not spent coding, but testing the rest of our functionality to make sure it wasn't slower.



 
