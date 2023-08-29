
Join Order optimization

- Preamble. The old join order optimizer was a mess. and most of it was written by me, which should explain how the mess came to be. So there you go.

There were two files, join order optimizer and cardinality estimator. If you know anything about join order optimization, you might be scratching your head right now. What about a cost model? What about relation extraction? What about plan enumeration? What about hypergraph construction? Well you guessed it, those logical components were all crammed into the join order optimizer. A fun 1000+ lines of code file that was nothing short of insane to debug.

### Join Order Optimization, what is it really?

After working on Join Ordering for about 1.5 years, I confidently say that join order optimization is not just reordering a query plan. In a naive way it is, but it really isn't. To start, let's inspect the phrase join order optimization. "Join order" sounds trivial, the order in which you join tables. "Oh, ordering, so it must be similar to sorting right?" is a logical step you may take. I'm here to tell you no, if it was, the problem would be called join sort optimization, and sorting algorithms are pretty advanced nowadays, and join ordering would be solved. 

So why is it ordering and not sorting? If you have a set of tables you want to join, once you perform a single join between two tables (or two sets of tables), the cost to join with another table changes. The cost of joining to tables is a function of all attributes of both tables. 

Let's start with a very very simple example. Suppose we have three tables; A, B, and C. With these three tables, we have 3 possible join orders. (A join B) join C, A join (B join C), and (A join C) join B. 


### Applying filters on nonreorderable operations.

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