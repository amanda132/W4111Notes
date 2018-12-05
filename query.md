# I. Introduction
## 1.1 Background
    Task: Find an efficient physical query plan (execution plan) for an sql query.
    Goal: minimize the evaluation time for the query, compute query result as fast as possible.
    Cost Factors: Disk accesses, read/write operations, [I/O, page transfer] (CPU time is typically ignored).

## 1.2 Translate Sql to query plan
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-15%20at%204.32.41%20PM.png?raw=true)
+    Query plan: A query plan (or query execution plan) is an ordered set of steps used to access data in a SQL relational database management system. This is a specific case of the relational model concept of access plans. It can be represented as a tree of relational operators. You can translate from SQL to relational algebra, or you can build the tree directly.
+    Parsing and Translating: Translate the query into its internal form/ parse tree (Given query may have many kinds of trees). This is then translated into an expression of the relational algebra. – Parser checks syntax, validates relations, attributes and access permissions.
+    Evaluation: The query execution engine takes a physical query plan (aka execution plan), executes the plan, and returns the result.
+    Optimization: Find the “cheapest” execution plan for a query.

### Notice:
    A relational algebra expression can be evaluated in many ways. 
    An annotated expression specifying detailed evaluation strategy is called the execution plan 
    (includes, e.g., whether index is used, join algorithms, . . . ) 
    among all semantically equivalent expressions, the one with the least costly evaluation plan is chosen.
## 1.3 What Optimization Options Do We Have?
+    Access Path 
+    Predicate push-down 
+    Join implementation 
+    Join ordering

# II. Access Path
#### 2.1 Example
<img src = "https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-12-05%20at%201.08.14%20AM.png?raw=true" width = "300">

- Push-based execution (op at a time)
  * read R
  * filter a>10 and write out 
  * read and project a
  * Cost: B + M + M

- Pipelined exec (page at a time): Read a page from offers and pass to selection. As selection looks for tuples in the page that satisfy the predicate, can read the next page from offers. Similarly, pass valid pages to projection, and as projection returns value of projected attributes, selection can look for tuples in the page that satisfy the predicate.
  * read first page of R 
  * pass to σ filter $a > 10$ and pass to $\pi$ 
  * project a (all operators run concurrently) 
  * Cost: B
  * Pipelining is usually cheaper than naive execution, because temporary relations are not generated and stored on disk. However, it is not always possible. For example, sorting, as each page needs to be compared to the rest unsorted pages.

- **If R is indexed** Hash index
  * Not appropriate
- B+Tree index
  * use a>10 to find initial data page 
  * scan leaf data pages
  * Cost: $\log_F B + M$.

## 2.1 Push vs Pull?
-    Push: 
   * Operators are input-driven
   * As operator gets data, push it to parent operator.
   * vectorization, batched computation
-    Pull: 
   * The root operator is likely the cursor
   * Operators are demand-driven
   * Do not do anything until parent operator asks for the next data.
   * pipelining

## 2.2 How to pick Access paths?
Access Path refers to the path chosen by the system to retrieve data after a structured query language (SQL) request is executed. A query may request at least one variable to be filled up with one value or more.

### Index + matching condition
+    Sequential Scan: doesn't accept condition.
+    Hash Index Search: accepts conjunction of equality conditions on all search keys.
+    Tree Index Search: accepts conjunction of terms of prefix of search keys.

Based on whether there is a "filter" operator directly above the scan operator, we can decide whether or not we want to use indices.

### Selectivity
+    Ratio of # outputs satisfying predicates vs # inputs
+    Assume attribute selectivity is independent
+    If attributes is primary key, you know exactly one matches it, and number of value equals to cardinality.

#### Example 2.2
Let: a=1 has 0.1 selectivity, and b>3 has 0.6 selectivity. The selectivity of a=1 & b>3 is 0.1* 0.6 = 0.06.

#### Example 2.3
+ Example of computing selectivity:
  + Ex1. A has 100 values: 
    + selectivity(A = 1) = 1 / 100 = 0.01
  + Ex2. A has 1000 values and 100 distinct values:
    + selectivity(A = 1) = 1 / 100 = 0.01
  + Ex3. A has 100 values, B has 10 values and C has 1 values:
    + selectivity(A = 1 and B = 1 and C = 1) = 1 / (100 * 10 * 1) = 0.001
  + Ex4. The range of A is [0, 50):
    + selectivity(A < 5) = 5 / 50 = 0.1
  + Ex5. A has 100 values, B has 10 values:
    + selectivity(A join B) = 1 / max(A, B) = 1 / max(100, 10) = 0.01

### System Catalog Keeps Statistics
+ The statistics info is kept as another table in most databases,
we can query this table like we query anything else.
+ System R is the first relational database. [See more about System R](http://www.webopedia.com/TERM/S/System_R.html)
+ Below is the statistics stored in System R, the information is not much because statistics in 1979 is very expensive, 
database today has much more complicated statistics information.
  + NCARD "relation cardinality” # tuples in relation
  + TCARD # pages relation occupies
  + ICARD # keys (distinct values) in index
  + NINDX pages occupied by index
  + min and max keys in indexes
+ Database will use this information to compute the selectivity, otherwise it'll use the default estimate (5%).


# Predicate Push down: 
<img src = "https://github.com/xz2581/project1/blob/master/8.png">

+ if I see (b), (b) can be transformed into (a) by pushing the selection operator down.
+ in a) we can combine scan data(R) and filter (a>10) together into index to find all the a > 10, but it is not applicable for b).

Which option is faster if we have a B+ tree index on a?
+ a)Log F(B) + M pages: using Btree, go down the tree and find the starting value for a and scan to the right
+ b)B pages : not using B tree, scans entire relation

## Projection with DISTINCT clause
need to de-duplicate e.g., π</sub>rating<sub> Sailors
						
Basic approaches

+ 1. Sort: fundamental database operation
  + sort on rating, remove duplicates by scanning sorted data
  + O(2n + n): 2n to sort and n to scan
+ 2. Hash: partition into N buckets remove duplicates on insert						
+ 3.Index on projected fields: scan the index pages, avoid reading data 


## The Join 
- Core database operation
 join of 10s of tables common in enterprise apps
						
- Join algorithms is a large area of research
  - e.g., distributed, temporal, geographic, multi-dim, range, sensors, graphs, etc
  - Three basic joins: nested loops, indexed nested loops, hash join
						
- Best join implementation depends on the query, the data, the indices, hardware, etc


### 1.Nested Loops Join
```		 	 	 		
# outer ⨝1 inner
# outer JOIN inner ON outer.1 = inner.1 
for row in outer:				
    for irow in inner:
       if row.attr == irow.attr:      # could be any check
                yield (row, irow)
```
#### Property: 						
- Very flexible,can implement theta join
- Equality check can be replaced with any condition
- Incremental algorithm
  - The join will generate record as you execute it, while some other join executors need to wait until you create the hash table or sort the data before it can start outputting results
- Cost: M + MN, pretty expensive
  - outer M pages; Inner N pages
  - For each row of outer, go through each row in the inner and check. If it is matched, then yield that.						
- Is this the same as a cross product? 
  - No, for cross product, just remove the predicate check.


#### What this means in terms of disk IO					
+ Ex. A join B
  + A is outer. M pages
  + B is inner. N pages 
  + T tuples per page
						
+ Cost: M +T × M × N
  + Scan through the outer once, costs M pages.
  + for each tuple t in table A, (M pages,TM tuples)
  + scan through each page in the inner (N pages) 
  + compare all the tuples in with t 


#### Join Order Matters
+ A join B: M + T x M x N      
+ B join A: N + T x N x M     
+ Note: 
  + If inner is small IO, can fit in memory, then cost is M + N.
+ You have to incur a quadratic cost for implement this join, ideally you want something linear, so we can try indexed nested loops join


### 2.Indexed Nested Loops Join					
```						
for row in outer:
    for irow in index.get(row[0], []):
	yield (row, irow)
```
+ For every row in outer, do index look up and only iterates through everything that matches.		
+ Slightly less flexible
+ Only supports conditions that the index supports 


#### What this means in terms of disk IO						
+ A join B on sid
  + M pages in A 
  + N pages in B
  + T tuples/page
+ Primary B+index on B(sid)
+ Cost of looking up in index is C<sub>1<sub> 
+ predicate on outer has 5% selectivity (if there is filter over A)
						
+ Cost = M + T × M × C1
  + M: read all the outer table in A
  + T × M × C1:for each tuple t in the outer, (M pages,TM tuples), incur the cost of looking up index
```
for each tuple t in A:
     if predicate(t):                 #5% of tuples satisfy predicate
        lookup_in_index(t.sid)
```
  + Cost is approximately M + T × M × C1 x 0.05


### 3. Hash Join
+ Type of index Nested Loops Join;
+ When no index on inner table A join B on sid
+ How: Create a hash table on inner loop and then run indexed nested loops
						
 + M pages in A
 + N pages in B
 + T tuples/page
+ Cost of looking up in index is C<sub>1<sub> 
+ predicate on outer has 5% selectivity
						
+ Cost = N + M + T × M × 0.05 × C<sub>1<sub>

```
# for every sid in B, create a key, and then match all the tuple with that particular sid. 
# By doing so, we can speed up C<sub>1<sub>
index = build_hash_table(B)                    #N pages
    for each tuple t in the A:                 #M pages,TM tuples					
         if predicate(t):                      #5% of tuples satisfy predicate
            lookup_in_index(t.sid)             #C1 cost
```
<img src = "https://github.com/xz2581/project1/blob/master/9.png">

#### Questions:
+ Q: Given a bunch of joins, which order do we use?
+ Q: given two tables and a bunch of indices, what is the best way to execute it?
+ A: Use cost model to decide join order and what the best execution for single join should be

#### Optimizing single join Example:
+ R join S on id
  + |R| = 1000 pages
  + |S| = 100 pages
  + Tuple/page = 100
+ For single join, go through all the combinations, because it is cheap enough to do this.
+ For this problem, we can do nested loops and hash join
<img src = "https://github.com/xz2581/project1/blob/master/12.png">



#### Optimizing joins within multiple tables:
+ Use Selinger Optimizer (ex. R⋈S⋈T)
					
### Selinger Optimizer 
Goals: don’t go for best plan, go for least worst plan
						
Two Big Ideas:

1. Cost Estimator
  + “Predict” cost of query from statistics
  + Includes CPU, disk, memory,etc (can get sophisticated!) It’s an art
							
2. Plan Space
   + Avoid cross product
   + Push selections & projections to leaves as much as possible 
   + Only join ordering remaining 
   + Try to reduce the possible trees to one that is manageable. 
							
### 1. Cost estimation
Given an operate, input and statistics, we should be able to estimate the cost
- Estimate cost for each operator
  - depends on input cardinalities (# tuples)  and data structure you have					
- Estimate output size for each operator
  - need to call estimate() on inputs!
  - use selectivity. assume attributes are independent

#### Estimate Size of Output
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-20%20at%204.22.50%20PM.png?raw=true)
+ To check if col = v, we assume uniform guess.
+ To check col1 = col2, the selectivity is min(ICARD_{col1}, ICARD_{col2})/ICARD_{col1} * ICARD_{col2}.
+ To check col > v, we assume all the values are uniformly distributed.

### Example
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-20%20at%204.29.02%20PM.png?raw=true)
						 	 	 		
- Naïvely
  - total records		1000* 10		=10,000
  - Selectivity of emp          1 / 1000		= 0.001
  - Selectivity of dep          1 / 10			=0.1
  - Join selectivity            1 / max(1k,10)  	=0.001
  - Output cardinality          10,000 * 0.001   	=10											
+ Note
  + Selectivity is defined with respect to cross product size
  + estimate wrong if this is a key/foreign key (ex: assumes that dept.did is the primary key and emp.did refers to dept.did, then join on emp.did = dept.did should yield 1000 results, not 10, because each emp.did has a corresponding dept.did. But when emp.did and dept.did are both primary keys in each table, the result should be 10, i.e. the smaller one, because the primary key should be distinct.)

### Example: switch orders
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-20%20at%204.33.36%20PM.png?raw=true)
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-20%20at%204.35.47%20PM.png?raw=true)


### 2. Join plan space
A⨝B⨝C 
Possible plans: 12
+ (AB)C (AC)B (BC)A (BA)C (CA)B (CB)A
+ A(BC) A(CB) B(CA) B(AC) C(AB) C(BA)

number of plans = number of permutation  * number of possible strings.

+ number of parenthetizations: choose(2(n-1), (n-1))/N
+ number of possible strings: n!
+ e.g. N = 10  number of plans =17,643,225,600
Note: The following two joins are not the same!
      + The "outer" table in these two cases is likely to have different cardinalities thus the two joins is likely to have different costs
<img src = "https://github.com/xz2581/project1/blob/master/13.png">


### If the plan space is too large,we can simplify the set of plans so it's tractable and do the following:  
1. Push down selections and projections
2. Ignore cross  products
3. Left deep plans only
   + only outer is allowed to have join, which means only left side is allowed to have subtree. The right side is always a leaf.
   + The reason for choosing left deep plan
     + it allows pipelining. If the AB can generate a tuple, then we can immediately start to join with the other table C while this is impossible for right deep plan because:
        + The inner in this case is B⋈C and the outer is A
        + If we want to join inner for every tuple in A, we need to re-compute the join or wait until the inner is completed. 
        + Also, if the inner is the output of the join operation, then we don’t have any indices.				
4. Dynamic programming optimization problem
  + Idea: If considering ((ABC)DE),figure out best way to combine with (DE)			
  + Dynamic Programming Algorithm
  compute best join size 1, then size 2, ... ~O(N*2N) 
5. Consider interesting sort orders  

### Reference Algorithm
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-20%20at%204.56.49%20PM.png?raw=true)

## Selinger Optimizer Example A⋈<sub>x</sub>B⋈<sub>x</sub>C⋈<sub>x</sub>D
### Preliminaries
+ We have primary B trees B(x) and C(x) both with height 2.
  + C = cost of indexing
      = height + #pages that matched the predicate (1 here since we have equality predicate) = 2 + 1 = 3
+ x values are distinct in each of the table A,B,C,D.
+ Cardinalities (in pages)
  + |A| = 100 
  + |B| = 1000
  + |C| = 10,000
  + |D| = 100,000
  + T = #Tuples/page

### Step One: Find the two-table joins with the least cost (Dynamical Programming example)
+ The existence of B tree indices B(x) and C(x) suggest that B or C should be the "inner" table for the two table join.
+ Assume we use indexed nested loops, the possible combinations of joins and their corresponding costs are: 

<img src = "https://github.com/xz2581/project1/blob/master/3.png">

+ Here we have a tie between AC and AB, let's say we decided to go with AC.

### Step Two: Find the three-table join with the least cost
+ Two options (AC)B and (AC)D
  + For (AC)B we can use indexed nested loop.
  + For (AC)D since there is no index built for D, we can only use nested loop.
  + Intuitively, (AC)B should have less cost.
+ To estimate the cost for the three-table join, we need to find the estimated output size (in tuples) of A⋈C
  + Denote the output size by |A⋈C| (This is in TUPLES!! NOT IN PAGES!)
  + |A⋈C| = selectivity * |A X C|
           = 1/max(1K, 100K) * (1K*100K) = 1K
+ (AC)B and (AC)D with their corresponding costs:
<img src = "https://github.com/xz2581/project1/blob/master/16.png">
+ (AC)B is indeed the one with the less cost.

### Step Three: find the four-table join with the least cost
+ Since there is no index built on D, we can use either nested loop join or hash join
+ The details for calculating the cost of nested loop join and hash join are left to the readers.


### The full process for determining the join orders is shown below
<img src = "https://github.com/xz2581/project1/blob/master/7.png">