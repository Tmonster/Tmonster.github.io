---
layout: post
title: Optimizers: The Good, the bad, and the "Does My System do This?"
gh-repo: duckdb/duckdb-web
gh-badge: [star, fork, follow]
tags: [Databases, DuckDB, Optimizers]
comments: true
---


<!-- An optimizer can improve query performance by more than 100x. This is not the case with 99% percent of queries, but what if it is the one query you run for every dashboard refresh? Would you still choose to run your handwritten query without optimizations? Query optimizers are so advanced now they can perform optimizations that humans cannot handwrite.

So why is it important to have a query optimizer for a DBMS or any kind of data analytics? There is one main reason. SQL (and most other query languages) don't let you express how a query should be physically executed. SQL is a declarative language at heart, which means you just write **what** you want, and not **how** you want to get it. This means it is hard to tell a database to perform a filter above a table scan in SQL, or to rewrite certain expressions so that a join condition becomes an equality condition. What you can easily do in SQL, however, is tell it exactly what you want, and that's what you will get. You don't have to mess with 20 different lines of code that carefully craft subsets of filtered data so that a join is less expensive. Write SQL to say what you want, and the optimzer (theoretically) tells the DBMS how to get it while maintaining logical equivalence to the original query.
 -->


Optimizers is a great candidate for a love-hate relationship for data analysts -- that is, if they know optimizers exist. An optimizer can improve query performance by 100x, but their con is that sometimes they are too smart for their own good and can ruin query performance. For most optimizers, the former occurs much more often than the latterm, but, for anyone who hath experienced the fury of  bad query performance due to the optimizer, let me attempt to persuade you to fall back in love with the optimizer.

<!-- This is not the case with 99% percent of queries, but what if it is the one query you run for every dashboard refresh? Would you still choose to run your handwritten query without optimizations? Query optimizers are so advanced now they can perform optimizations that humans cannot handwrite. -->

Why is it important to have an optimizer for data analytics? In my opinion it comes down to language and communication, specifically how we communicate with our database systems between what we want and how we want it retrieved. Let's take a very simple SQL query. 

There are two classes of languages that can express this. SQL, or declarativie languages, and dataframe libraries, which you can also call imperative languages. 

With SQL, we could have a query like the following

```sql
select 
	max(store_revenues.revenue), 
	store_revenues.region, 
	personnel.employee_name
from (
	select
		sum(sales) as revenue, 
		region, 
		manager_id 
	from store_sales
	group by region, store_id
) store_revenues, 
personnel,
regions 
where 
	store_revenues.region = regions.region and
	store_revenues.manager_id = personnel.id;
```

This is a little complex, but still readable. We can see that we need to query from the store_sales table, the personnel table and the regions table. What's great about declarative languages is that we only need to write **what** we want. With imperative languages, however, we also need to describe **how** we want to get it.

Here is an equivalent snippet to get this information in imperative languages.

```R
# todo, dplyr code
```

```py
# Omitting setup to a db engine
# todo, datafusion code
```


This is much more wordy, and can be prone to more human error because it requires translating what you want, and how you want to get it.

### No more need to describe "how" you want the data.

If you are using a system that supports SQL, you get to benefit from the fact that it is a declarative language (usually, sometimes you can the database exactly how you want data stored or how you want something else done). You can just say what you want and be done with it.  The optimizer will take your statement, and figure out how to translate what you want into how the db-engine shoud get the data. Some imperative languages also have optimizers that will optimize your query, but by removing the requirement to describe "how" you want the data, you have to write less code, which (hopefully) means less bugs.

### So, but what is the optimizer doing that I can't do?

In my opinion, there are three categories of optimizers. Some optimizations are easy to communicate to the query engine and always improve query performance, optimizations that can be written by hand, but need to be re-written when the data changes, and optimizations that are impossible to express without language-specific functionality to the express the optimization.

#### Hand optimizations 

Here is a list of optimiziations that you can do by hand on your queries. Keep in mind that by doing theses optimizations by hand, you run the risk of introducing bugs into your code.

Filter pushdown.

##### Expression rewriting

This is a simple one. Suppose you have a join condition like `select * from t1, t2 where a + b = 50`. This could be written as `select * from t1, t2 where a = 50-b`. Rewriting the expression in this way allow the planner to plan a hash join instead of a nested loop join. To the human eye, however, `a+b=50` might be nicer to write because it can communicate the combined age of two players. 

##### Regex->Like Optimizations

Suppose your queries involve regex matching in order to match the prefix or suffix of a string. I.e 
`select * from t1 where regexp_matches(a, '^prefix.*)`. This is a simple prefix lookup, and without an optimizer, a regex library will be used to run the prefix check. Certain db-engines have optimized prefix lookups and can perform them 100x faster than a regex library. Again, this is possible by hand, but it's nice when this happens automatically.

##### filter pullup and pushdown

Suppose you have a query like `select * from t1, t2 where a + b = 50 and regexp_matches(c, '^prefix')`. Filter pull up and pushdown will automatically convert the `a=b` filter into a join filter, and will push the filter on column `c` into the scan of t2 instead of performing it at the top.


Lets take a look at the two query plans with one taking advantage of each optimization, and the other query that is not optimzed.

#### Optimizations that specific to data state

The optimizations below are possible by hand. If the state of the data changes, however, it's possible that the query performance becomes worse by a factor of 100x.

##### Join Order Optimization

Let me try to summarize my MSc thesis in one paragraph. When executing any query, it's important to avoid keeping large amounts of data in memory. For join heavy queries, predicting the size of the join output can be difficult, and if we predict a small join output when in fact it is very large, query performance can suffer since handling large intermediate data is slow. Join order optimization attempts prevents this by looking at the stats of a table and attempting to predict the cardinality of joins to prevent an exploding join producing an unmanageable amount of data. Join Order Optimization is possible by hand of course, but when the underlying data changes, it's possible the intermediate join outputs increase, which can lead to poor query performance.


##### Statistics Propagation
##### Build-Side Probe Side Optimization

Certain 

An optimizer can perform these optimizations with up-to-date knowledge of the data every single time, so the optimization is valid for almost every state your data is in. If an analyst were to hand optimize a query, and later dump a huge amount of data into certain tables, query performance would significantly decrease, which could cause a number of problems for downstream processes.


#### Requires specific language 

These optimizers below are, in my opinion, impossible to describe without an optimizer.

##### Top N optimizations 
This is a relativly simple optimization. If you have some sql like `select a, sum(b) group by a order by a asc, limit 5` there is room to optimize this to only keep track of 5 aggregate values. Which means you won't need to store the result of `sum(b)`  for the rest of the values of `a`. 

##### Join Filter Pushdown
This is an optimizer in DuckDB that can dramatically improve the performance of hash joins and related table scans. During the hash join process, the keys of one table are stored in a hash table. Join Filter Pushdown will store the min&max values that are stored in the hash table. Then, when the probe side is executed to start checking matches, the table scan at the bottom of the probe first performs a zone map filter on the data it is scanninig to see if each tuple is within the min and max values in the build table. Confused? Imagine explaining in SQL.

##### Table filter scans











<!--  Well it comes down to how we as humans translate our desired data result into a language an analytics engine (or transactional engine if you use postgres) can understand. For this blog post, let's compare SQL as used in DuckDB with a dataframe library like pandas. 

With SQL, you can declare what you want, making the translation between desired data to machine readable query easy. If I want some average of all sales grouped by region, my query could look something like this. -->

What's nice is that SQL is declarative, and I can just write in SQL, exactly what I want. If I used pandas, I not only have to write **what** I want, but also **how** I want it retrieved.

```py
# TODO
```

With this kind of a script, you can also run into issues around eager and lazy evaluation, but that's a separate blog post. When I look at these two examples, I see the following reasons as to why an optimizer is so important

- There is more human error when a human must explain **what** and **how**
- Optimizers can tell the db-engine **how** to get the data much better than any human
- Optimizers can optimize in ways a humans cannot explain easily
- An optimizer can change **how** the data is retrieved based on changes in the data

Why I think an optimizer is so important, is because it can tell the db engine **how** to get the data better than any human can, and in ways that we as humans can not express ourselves without using 


Well, there are a few reasons. Sometimes the general physical makeup of the data can influence the performance of a non-optimized query. Other times, changes in the make up of the data can render a previous implementation of handwritten ETL code innefficient. And finally, some logical optimizations are impossible to write by hand.


<!-- ## Some Examples

We will use the tpch dataset for this problem. To compare the performance with optimization and without optimization, each scenario will be performed with the optimizer on and with the optimizer off. The queries will be written specifically to show the important of one optimizer. (yes, there are multiple optimizers).
 -->

### You can't express physical execution in SQL


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

### Data can change, which means the query should change (physically)

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



 
