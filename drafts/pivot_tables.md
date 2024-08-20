why pivot tables. 

TL;DR Pivot tables are used to store data because they usually represent some aggregation of data over a time frame. I.e total or average sales of an item in one month/day/year. When the aggregation or the time ointerval changes, the data stored would need to double.


Suppose you are tracking the number items sold per item. You need to choose between storing it in a pivot table, and storing it in a "sold items" table. Here is what is needed when turning it into a pivot table. **Warning, this is a blog post to tell you why you should not store data in a pivot table**


### PIVOT TABLE

- Each row keeps track of an item
- Each column stores a month
- Each cell now stores the number of times that item was sold every month

Easy right? What will you do when...

- You want to see the number of items sold per day? per hour?

Then you need to duplicate your table.

What happens if you want to add items?

- Then you need to add a row, but then you also need to add values for all timeframes recorded

- Suppose you want to add an item. Easy, you add another row for the item. But for every column you have to input NULL
- Suppose you want to add a new time interval, then you have to add a new column for this. If your time intervals are short, you may end up adding a lot of columns that don't have values, while other columns do have values





