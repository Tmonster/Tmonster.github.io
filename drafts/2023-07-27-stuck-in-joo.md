
Join Order optimization

TL;DR. I refactored the join order optimizer in duckdb (which I also originally wrote for my Masters thesis). Below are just my findings summarized into (hopefully) 500 words or less.

## Let's start with an example

Let's take a look at three simple examples of join order optimization. These examples can explain the power of good join order optimization given the correct operators. Each example will also be references one or more times later on in this blog post.

Example 1. 
Simple A x B x C (cardinalities 10, 100, 1000)

Example 2. 
Left joins & right joins (they are not reorderable)

Example 3. 
Aggregates and window functions. You cannot reorder tables up through the operator.

Example 1 shows how powerful join order optimization can be. Examples 2&3 show some of the challenges. But how can we deterministically choose a good join order? There are many papers that touch on join ordering, and you can feel free to read them if you like. To summarize all of them however, it comes down to having good cardinality estimation. This is practically proved in (how good are optimizers really?) and (my thesis). (my thesis) is where the rest of this journey starts.

## The Original Organiziation

There were only two files responsible for all the logic to perform join order optimization; join_order_optimizer.cpp and cardinality_estimator.cpp. These are two necessary and obvious components to Join Order Optimization, but we can break down join ordering into more logical components, each of which arguably deserve a separate file. So let's take a look at each logical step, and see how it fits into a join order optimization workflow.

### Join Order Optimization, what is it really?

After working on Join Ordering for about 12 months, I can confidently say that join order optimization is not *just* reordering a query plan. In a naive way it is, but if you know what you are doing, it really isn't. Join Order is not reordering a query plan, because it involves the following extra logic

1. Extracting Relations & Filters
2. Creating a query graph
3. Enumerating potential plans (keeping track of the cost of each plan (see _cardinality estimation_))
4. Re-building the plan
5. Build-side/probe side optimizations.


### Extracting Relations & Filters. What is reorderable, what isn't.
When we start join order optimization, we are usually presented with a logical query plan. The logical plan can have operators like joins, filters, group bys, and aggregates. 


### Creating a query graph and understanding what a hyper graph is

### Enumerating potential plans Dynamic programming style

### Rebuilding the plan

### More optimizations

### Applying filters on nonreorderable operations.


## Conclusions


#### So what are we reordering exactly?
1. first you need to find reorderable relations and non-reorderable  relations
2. Don't forget that left, right, and outer joins exist.


### What does the new Join Order Optimizer look like?

1. Relation extraction, you don't want to directly reorder your logical plan, you need some way to treat every node as some hashable structure so that it can be placed into a dynamic programming table. This step is already hard because you need to determine what relations are reorderable and what relations are *not* reorderable. For example, inner joins are reorderable, while left joins are not. 

1. (B) While doing relation extraciton, you need to keep track of what the join filters are. 

2. Statistics extraction from every relation. Once you finally understand what operators are reorderable and which ones aren't, you need to create a relation for the operation. You might be thinking, "Ah, this is what we use for our dynamic programming table". Actually, no. We will use *JoinNodes* for that. Currently, SingleJoinRelations are used to store the reorderable operation, some statistics about the reorderable operation, and the relation id. (But tom, why don't we just create the join nodes directly from the operators, why do we need this intermediate structure.) I'm so glad you asked, because step 3 still needs the intermediate structure.


3. Now you can try to create your hypergraph. You have the single join relations, and the relation edges. You need to the single join relations so that when you iterate through the join filters, you know what single join relations are involved in the join filter. This way you can create the edges. 

4. Now you have a hypergraph (hopefully you made it correctly). Now you can enumerate through all the join plans. As much as I would love to type out the dynamic programming strategy to do this, I'll just refer you to the incredibly dense join order optimizer paper located here.

4. (B) What the paper doesn't touch on, however, is how to estimate the cost of join nodes and how to know if these costs are accurate or not. Before even starting join enumeration, you need to initialize the leaf join nodes. Let's suppose the cost model requires knowing the cardinality of joining two join nodes. Well guess what, remember steps 1-3, and how we created join nodes from single join relations from the original operators? Good thing we extracted our stats from the operator when creating the single join node. This stats object holds distinct column value counts and estimated cardinality of the relation. DuckDB uses only these statistics to find the optimal join order.

Also, it's important to note that stuff happens

5. Suppose the dynamic programming strategy worked (which sometimes it doesn't), now you have a tree of join nodes, and you need 