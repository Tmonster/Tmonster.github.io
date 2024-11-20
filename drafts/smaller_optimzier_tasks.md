Optimize case expressions.

Maybe moving an always true to the else is not super efficient? It should be the left most expression no?

push down aggregate operations? Like what was suggested in the blog post?
Would require knowing FK-PK relationships though.
I.e in the taxi case, what happens is you
1. group by pick up and drop off locations
2. Join with zone_lookups on pickup locations. If there isn't a match, that's fine. After all there could be missing matches because certain zones are not in manhattan. If there are two matches, then you are kinda boned, because now the result will contain the aggregation row twice. 
3. 
