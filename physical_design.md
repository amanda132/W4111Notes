
# 1. Introduction
* Why do we care about hardware?
* What properties that really matter for design the rest of the system?

## 1.1 Work from bottom up
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-12-04%20at%209.26.46%20AM.png?raw=true)
- There is a total of 5 layers. Each of which is considered a separate component in the database management system software, and it provides a simple abstraction to layers above it, and makes assumptions about layers below it. 
- Query Optimization and Execution: This is the layer that processes the query we wrote. It's parse it, checks the syntax and verify the SQL query. For example, it checks whether the table to the in the from clause exist or not or any typos in the query. Then comes all the possible relational plans, which are different ways of ordering the relational algebra. The optimizer finds the best plan base on the time cost.
- Relational Operators: This layer takes the query from the above layer and implement the necessary relational operator. This layer is also called the execution engine, because execute the query above.
- Files and Access Methods: This layer organizes the tables, indexes as groups of pages in a "logical file". All the layers under this layer only works with pages because they are more easily manageable. There are many ways to organize pages; heap files are mention below.
- Buffer Manager: This software layer that is responsible for bringing pages from disk to main memory as needed. (p232)  In addition, there is a limit on how many pages this manager contains. 
- Disk space manager : The lowest layer of the DBMS software manage space on disk, where the data is stored. Disk space manager supports the concept of a page as a unit data, and provide commands to allocate or deallocate a page and read or write a page. Higher layers allocate, deallocate, read, and write pages through (routines provided by) this layer.(p231)

#### Example 1.1
When a SQL client enters:
```
db.execute(’’’
SELECT a, b
FROM S, T
WHERE S.c = T.c’’’)
```
- Query Parsing and Optimization: Parse, check, and verify the SQL query; turns the request into efficient query plan
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-12-04%20at%209.08.02%20AM.png?raw=true)
- Relational Operators: A query is represented as a relational data flow; each operator is a specific implementation
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-12-04%20at%209.10.45%20AM.png?raw=true)
- Files and Access Methods: Organizes tables, indexes, records as groups of pages in a “logical file”; 
API:
• Operators ask for records
• Logical files help read and write bytes on pages
- Buffer Management: Not all pages can fit into RAM, and buffer manager provides illusion that all pages are accessible; files simply ask for pages.
- Disk Space Management: Physically read and write bytes on one or more storage devices (hard drives, SSDs, etc); storage performance properties dictate the design of layers above



## 1.2 Disk vs RAM
Data is literally driven by money.
### Why Disk is important? 
1. Disk is the cheapest per Gigabyte Storage mechanism in the market
2. The process of analyzing and optimizing disk is the same process you might go through for optimizing any others
3. It allows us to retrieve any page at a (more or less) fixed cost per page. However, if we read several pages in the order that they are stored physically, the cost can be much less than the cost of reading the same pages in a random order.

### Why not store all in RAM?
1. Cost too much:
High-end Databases today store Petabyte (1000TB) of data
60% of the cost of operating a database is in supplying, managing and maintaining disks.
2.  Main Memory not persistent; It is important if DB stops or crashes. 

#### Example 1.2
If you spend $ 1000 on hardware, you can get 64 - 96 GB for RAM, 400 - 1000 GB for SSD, 24000 for Disk. 
You can spend some money on RAM for active data, Disk for main database, secondary storage, and Tapes for archive.
- What does this mean? 
 You can prioritize where your money goes to- ram, disk, etc.

### What ends up with is following architecture:

![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/layers.png?raw=true)

- You will have most of your storage capacity on disk, because it’s cheap.
- As you go up the memory hierarchy, things become more expensive but faster. 
- Register is a piece of memory that is accessible in CPU. (Ridiculously expensive)
- You can purchase RAM with reasonable prices.
- The faster you go up the less storage space you have. You need to optimize the rest of the system. 

### Interesting numbers:
1. The cost of compression 1k bytes is 3000 ns. This means if you can compress your data sufficiently, that is more worthwhile to do than storing the uncompressed version in disk. It is faster to decompress the memory than to fetch the uncompressed version from disk.
2. If you run a data center, the cost of access data in memory from another machine is .5 ms, which is significantly cheaper than disk.

- The above two examples of some of the things that influence design decisions. What does this mean? If you can afford multiple machines with lots of RAM, it makes sense to use Disk for other things you never access, and put everything on other machines. 
- Jim Gray found the idea of transactions, recovery, and other core databases ideas. Below is a storage hierarchy analogy. Going to memory/RAM is like driving to Philly and then coming back. Going to Disk is like going to Pluto and back. (“You might as well give up on Tape.”- Prof Wu.)It is important to minimize the time you need to access data.

![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/jimgray.png?raw=true)

# 2. Disk
##  What is a disk? How a disk work?
- Think of the disk as a very fast DVD or record player. Stack a bunch of the round disks on top of each other. The tip of the head will know how to read and write whatever is underneath it. This is how DVD and record players work. Each of the rings correspond to a track and it stores data. Each track is split into segments called Sector. 

![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-12-04%20at%209.54.16%20AM.png?raw=true)

- This a representation of hard drive. The tip of Head know how to read and write what ever underneath it. Each of the ring is a track, which stores data. Each track is separated into several sector.

### Interesting properties:
1. The size of the sector is determined by the rotational angle rather than a fixed length. So the sector far away from center will be larger than the sector near the center. You can read more data outside than inside for a given amount of time.
2. How much data you can read depends on the spin speed. You want to maximize the RPM.
3. If you want to move the arm in/out to read some data, there will be a seek cost (moving the arm to access data). It dominates by far the cost of accessing data.

![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/timetoaccess.png?raw=true)

- Problem is the first 2 delays if you care about speed (the latency in which to get the first bit of data to read). We want to optimize seek and the rotational delays. Seeks are expensive, but reading things that is right underneath the head is very fast. This is called sequential access.
- Sequential access: access to a computer data file that requires the user to read through the file from the beginning in the order in which it is stored.
- Seek cost: the cost for moving arm.
- Seek time : the average time to move arm
- Rotational delay: the data you want is not directly underneath the reader, so you need to wait until the disk rotates until you can actually read that thing.
- The key thing here is to reduce seek and rotational delays: HW & SW approaches. 

## Pre-fetching
We want to optimize the amount of data we can sequentially access. Disk drive can do pre-fetching. If you are sequentially reading some data in a file that store sequentially on disk, then disk driver can just start reading ahead. E.g. doing data processing and pause it, next time you read the next data, it will already be there because it has already fetched it for you. 

## Graph of read cost for random vs. sequential access
If you are doing random access (randomly placed in storage device), how many can you read per sec? 316 values/sec
If you look at memory, you will see that there is higher throughput. 
Random access between memory and disk is pretty much on par. 

![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/valuereadpersec.png?raw=true)

## Pragmatics of Databases
- Most databases are pretty small
- All global daily weather since 1929:20GB
- 2000 US Census: 200GB
- 2009 english wikipedia: 14GB (you can fit this in your laptop!)
- You don’t have to have Google infrastructure to do interesting things. 

## What is the API between data base system and disk?
- API is centered around a page, a fixed size block of data. This is the unit we pass around. We want to amortize the cost of having to move that arm. 

- If page you read from disk too small: dominated by moving arm around
- If page is too big (ie. 1 GB in size): read a huge amount of data but only access a small amount inside of it. Typically Operating system will pick a page size between 4kb and 64kb in size to balance out moving the arm and reading more data than you ask for. 

- The data we store is in terms of pages. 
- On disk, we store a file that represents Customers Table. We split and store Customers Table as pages. 

![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/table.png?raw=true)

## What is a page?
- Unit of transfer between storage and database
- Typically fixed size
- Small enough for one I/O to be fast
- Big enough to not be wasteful
- Usually a multiple of 4 kB 
- Intel virtual memory hardware page size
- Modern disk sector size (minimum I/O size)

### Default page sizes in DBs
Note: Typically multiple of 4 kBs
* SQLite: 1kB
* IBM DB2: 4kB
* Postgres: 8kB
* SQL Server: 8kB
* MySQL: 16kB
* MongoDB: 32kB

### Strategies for Fast Data Access
* Amortize: sequentially read & write big chunks of bytes
* Cache popular blocks
* Pre-fetch what you will need later

### Disk Space Interface
Below is the API. There are 4 ways of access things.

DiskInterface: API, four ways of accessing things
* `readPage(page_id): data` give it a page id, database will translate that into a position on the disk drive
* `writePage(page_id, data)` do the same thing as readPage, but also provide a data
* `newPage():page_id` allocate space on disk, give me the `page_id` of that 
* `freePage(page_id)` I am done with this page, clears up space, so it can be reused.


### Record, Page and File Abstractions
![](https://github.com/amanda132/W4111Notes/blob/master/file_page.png?raw=true)
* File is a collection of pages for which if you collect all the pages together, you get all the data. 
* Everything is going to be stored as files. 
* A page is  “collection” of records because we can’t keep track of the order. Keeping track of the order is not needed as a guarantee. However, it is useful to have sorted pages to be able to perform binary search.
* Record: "application" storage unit
  * e.g. a row in a table
* Page: Collection of records
* File: Collection of pages
  * insert/delete/modify record
  * get(record_id) a record
  * scan all records
* May be in multiple OS files spanning multiple disks

### Units that we’ll care about
- B # data pages on disk for relation
- R # records per data age (we talk about it in terms of records, not the physical size of page) 
- D avg time to read/write data page to/from disk. 

- I.e. if i have 100 records per page, and 100 pages in a table, then we’ll have roughly 10,000 records in that particular table. 
- We ignore sequential access for now. It will be added back in later on.

## Different ways to store a table
- Criteria: accessing, deleting, inserting data
- Unordered Heap Files
- Unordered collection of records
- Pages allocated as collection grows
- Need to track:
 - Pages in file
 - Free space on pages
 - Records on pages (which pages are free, which pages are full, etc)
 - Example: let's say you have a table with attributes (a int, b int).  Then you might store (say) 1000 (a, b) value pairs in one page.  One question is how you store them within a page --- sorted?  unsorted?  A second question is how to organize the pages in the table.  You may not be guaranteed there's contiguous sectors on the disk drive for all of the pages so some pages won't be next to each other.  So they will need to be able to "point" to their neighboring pages.  One way is to organize the pages as a linked list.

### A data page is just a bunch of records + two pointers 
- Each data page as a pointer to the next and previous data page
- If we want to read and write data, we will start at the first page, and read on until the record I care about. 
- Since this is unordered, looking up records will still require reading potentially all the pages. This is the dumbest way of representing it.
- Smarter way: keep track of which pages have full data and which pages have free space. (as shown in the image below).

![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/heapFile.png?raw=true)

### What might be a smarter way of doing this? 
- We can have pointers to all pages. Instead of single header page, we use a directory. 
- You keep track of two numbers: the data page it points to, the amount of free space. 
- Small constant cost to reading the directory to find which pages are free if you want to insert data
- Note that there is there are multiple data page rather than one is because when a record is removed from a full data page, the pointer of that page is reassigned to the end of the data page with free space.
- Even smarter: we can ensure the pages are sorted. In order to do that, the directory also has to keep track of the minimum and maximum value of all the data in each page. This will allow us to do binary search faster. Slightly faster if we start storing more information in the header page. 
- If you want to search for specific records, this way still requires you to read everything . 
![](https://github.com/pyw2102/w4111ScribedNotes/blob/master/Physical-Design/directory.png?raw=true)

##L20: Disk Storage, Heap Files, B+ Trees, Hash Files, Single Operation Optimizations

## Pre-lecture Primer
- **Question:** How can we access data quickly?
- **Thought:** We need different options with different trade-offs to compare.

![](https://github.com/shy2116/project1/blob/master/disk.jpg?raw=true)

- **Visual:** Disks have tracks, and data is stored on sectors along a track. An arm then reads this data. Sectors are roughly parallel to the concept of pages, the unit of data transfer and manipulation we will be using.

- **Basic scenario V1:** We store data from tables in these pages at random, and assign a pointer to a page from which to start from.
 - Q: Is this acceptable?
 - A: No, we need a way to move from one page to another.

- **Basic Scenario V2 (linked list):** Let’s add pointers from one page to another and a backward pointer for good measure. 
This is functional, but we have to read through all the data to find something
- **Basic Scenario V3 (sorted linked list):** All the data along the linked list is sorted by a particular primary key, optimized for a specific query.
 - Q: Can we run a binary search on this? No.
 - Q: How do you find the middle page? You can’t.
We need something to tell us where the pages are and what they contain without having to read all the data.

- **Basic Scenario V4 (directories):** Let’s create an “upper level” set of pages that serve as directories pointing to these pages
 - Q: What if we have a billion pages in the lower level, and we are only able to narrow it down to a million “upper level” directories?
 - A: We can recursively create additional levels of directories
 - Additional observation: We can use directories to find a starting point, and once we get to a page in a sorted list, we can scroll through without having to go back up the tree.

- **Basic Scenario V5 (Alternative directory):** Instead of pointing to pages, we can point to records
 - This is useful if we don't want to deal with sorted data, which can be difficult to maintain

## Indexes
### Reason for using index
- **Idea:** If you know you’re going to do something often, it’s worthwhile to make sure it can be done fast
 - Abraham Lincoln: "If I had eight hours to chop down a tree, I'd spend six sharpening my ax."
 - Eugene Wu: "If I had eight hours to chop down a tree, I'd spend six organizing my indexes." Fall 2018
- **Point:** Creating and storing indexes on pages takes up additional space; however, the amount of space is relatively small compared to the database it's based on, and the queries they are tailored to can execute infinitely faster.
- **Offline (vs. online):** Allowing a database to prepare all the data structures and encoding it needs, so queries can run fast and efficiently
- **Two types of queries:**
 - Equality query: An attribute equals something
   - SELECT name FROM sailors WHERE sid = 11111
 - Range query: An attribute satisfies a set of inequalities
   - Select name FROM students WHERE age > 19

### Some knowledge about index
- Indexes are defined wrt a search key, an attribute or a set of attributes that you want to be able to query efficiently. - **Naming collision:** A search key is different from a candidate key! A search key does not necessarily identify a unique tuple, it can be on any attribute, such as age or sex in a database containing information on everyone alive on Earth.
- **Application:** Search keys can provide faster access for WHERE clauses

- In SQL, you can use the following syntax to assign an index for to a table. The CREATE INDEX command creates B-tree indexes by default, which tends to be appropriate for most situations

`CREATE INDEX [idx1] ON users USING btree (sid)`

- **Valid queries** using this index
`SELECT name FROM users WHERE sid = 11111`
`SELECT name FROM users WHERE sid > 11111`
`SELECT name FROM users WHERE sid > 11111 AND age > 18`
- The index key is on the attribute sid, so the index is specially tailored to queries with sid in their WHERE clauses.
- In the third query, although the index is not on age, having an index on sid can significantly reduce the number of tuples that must be checked. 
- **Invalid queries** using this index
`SELECT name FROM users WHERE age = 18`
`SELECT name FROM users WHERE age > 18`
- Because sid does not exist in their WHERE clauses, the idx1 cannot be used for these queries. If no other indexes are available, we can do no better than a full table scan.

![](https://github.com/shy2116/project1/blob/master/SQL%20index.PNG?raw=true)

- The database optimizer makes a cost analysis of the different indexes available and selects the method it deems the most efficient.
![](https://github.com/amanda132/W4111Notes/blob/master/Screen%20Shot%202018-11-13%20at%204.24.44%20PM.png?raw=true)

### High Level (Primary) Index Structure
![](https://github.com/shy2116/project1/blob/master/High%20Level%20(Primary)%20index%20structure.PNG?raw=true)
- The index and data entries (corresponding to index pages and leaf nodes) in the illustration make up the index.
- In a "Primary Structure," the actual data records (entire tuples of a table) are stored in the leaf nodes of the index, stored on the index key (search key).
- This structure has two significant page I/O cost implications:
 1. The index file requires more pages to store the actual data records in the leaf pages, meaning the height of the tree could potentially be taller than the secondary structure.
 2. The data records stored in the leaf nodes are sorted and clustered by definition. This means that when a query has multiple matches corresponding to an equality condition, all the matches are conveniently clustered in the leaf pages for us to access. The number of data pages (leaf pages for the primary) we will access a maximum of: (# matching tuples) divided by (# tuples a page can hold). For a query with a handful of matches, this is much more efficient than having to access a distinct page for each matching tuple.

### High Level (Secondary) Index Structure
![](https://github.com/shy2116/project1/blob/master/High%20Level%20(Secondary)%20index%20structure.PNG?raw=true)
- As opposed to the "Primary Structure," the "Secondary Structure" separates the indexing from the data records. This means that both the index pages and leaf pages (illustrated as data entries) serve as directories, where all entries are of the form <search key, pointer>. The index pages point to lower level index pages or leaf pages, and the leaf pages point to the record pages (data pages) containing the actual tuples corresponding to our search keys.

- This structure is a parallel to our Basic Scenario V4 (directories)
- Because the leaf pages only contain pointers, as opposed to actual records, which are almost always smaller in size than actual records, a leaf node can hold many more data entries. This has 3 important page I/O cost implications:
 1. The index file requires relatively less pages with a secondary structure, allowing for the height of the index tree to potentially be shorter.
 2. Because the leaf pages only contain pointers, an additional page access is required to to follow this pointer and access the data page containing the associated tuple.
 3. Assuming a reasonably high selectivity (low % of matches relative to total number of records in file), each pointer can be expected to point to a distinct page, requiring a page access for each qualifying record. Refer back to the cost advantage of using the primary structure.

Because we use only pointers, a page can contain many more pointers, but need to incur an additional cost to access actual records.

###B+ Tree Index
![](https://github.com/shy2116/project1/blob/master/B.PNG?raw=true)
- Everything is stored as pages (i.e. index pages, leaf pages, data pages)
- Index pages contain values of the index key and pointers to child nodes
- Leaf pages contain the data
- Next/previous pointers exist to perform range scans, as opposed to going through the index each time
 - For example: Provided: idx1(age) and Query: SELECT name FROM users WHERE age > 20
 - Once we have arrived at the leaf node corresponding to the leftmost leaf node satisfying the range condition, rather than going back through the index, we can use the next pointers between leaf nodes to scan right. 
 - The fact that B+ Tree indexes are sorted on the index key is the root of their significant advantage in evaluating range queries.
- Both equality and range queries can be used
 - Additional explanations and examples above
- Self-balancing: The distances from the root to the deepest node to the highest node differ at most 1
 - Implication: Amount of time to search is logarithmic to the amount of data you have (height)
- Disk optimized: indexes are optimized for specific types of storage
- Built bottom up: When you construct the B+ Tree, you need a starting point on the disk and the pages need to be sorted. - Given these two factors, it is much more efficient to build the tree bottom-up than inserting each record into the tree in a top-down approach.

**Basic B+ Tree: search key <age>**
![](https://github.com/shy2116/project1/blob/master/B0.PNG?raw=true)
- Note: This is a simplified example. The significant advantage of Btree indexes over the binary search alternative is that Btree indexes can have significantly higher fanouts (typically >100 rather than the 3 used in this example)
- The bottom nodes are the data pages that contain the actual tuples. In this example, the tuples are simply the index key.
- The index has at most 3 pointers corresponding to 3 conditions
- The index page has two index key values, 17 and 50. They are used to decide which child to follow when performing a search.
- **Example: ** SELECT * FROM file WHERE key = 16
 - We start at the node, and evaluate our condition against the conditions corresponding the three pointers. With the two index key values, 17 and 50, serving as dividers for the pointers. The first, second, and third pointers corresponds to the following conditions, respectively: values < 17, 17 < values < 50, values > 50. Because our condition falls within the range of the first condition, we follow the first pointer. We read from left to right, first evaluating the key 14, which does not equal 16. We proceed right to the next search key (we can do this because a Btree is sorted on its index keys), and find that the key search key, 16, does in fact match our condition. We return this tuple, and continue scanning right to check for any additional matches. Because leaf nodes in a Btree index are connected by previous/next pointers, we can proceed to the next leaf page without having to go through the index again. Reading the next page from left to right, we determine that the next next key, 17, does not satisfy our condition, and so we end our search.

**Full B+ Tree with additional record pages**
![](https://github.com/shy2116/project1/blob/master/B1.PNG?raw=true)
- The INDEX page is full, so we cannot add more child pointers to point to the two leaf nodes on the left. An additional index page must be added.

![](https://github.com/shy2116/project1/blob/master/B2.PNG?raw=true)
- We have added an additional index page to cover the two additional nodes; however, in doing so we also had to create an additional level and an additional page to point to the two index pages below it.
 - This is necessary, because we need a new root node to navigate the multiple index pages we now have.
 - The height of our index has increased by 1 to a value of 2.


**Composite search tree (multiple search keys)**
![](https://github.com/shy2116/project1/blob/master/Bage.PNG?raw=true)
- A search tree sorts in order of search keys from left to right
- Trade-off: Less entries in a given directory page, but you can answer queries based on two attributes.

**Query 1: SELECT age WHERE age = 14**
- Query finds that the age of the index is greater than the 14 we are looking for, so it knows to go left. The query finds the 14 it is looking for and returns the record. The query then moves on to the next record, find that’s 17 is greater than 14 and stops.

**Query 2: SELECT * WHERE age < 18 and name < ‘monica’**
The indexing order is first by age, then by name. Starting at (17, norton), the query finds that 17 is less than 18, and goes right to find the rightmost value that satisfies this constraint. Scanning through from the left of the node, the query examines (17, norton) to find that 17 is less than 18, so the query continues to go right in order to find the boundary. Upon examining the next record (24, alice), the query finds the boundary for the first index, as 24 is not less than 18. The query then goes back to the last valid record (17, norton), and examines the second constraint to find that ‘norton’ is not less than ‘monica’. The query proceeds left and finds that (17, alice) does satisfy the constraint. The query then returns the record and that of all the records to the left. 
Thoughts on efficiency: In a “Secondary” index, each pointer is potentially a disk access. The use of a “Primary” index allows us to filter out all the data we don’t need. If instead of (SELECT *), we chose to (SELECT age, name), we wouldn’t have to read anything beyond the index.

**Query 3: SELECT age WHERE name = ‘bobby’**
We can’t use this index because there is no criteria on age. We cannot do any better than simply reading through all the index data; however, considering that we are selecting an index value and the index is much smaller for a composite tree than the entire data set, this could potentially still be much faster.

![](https://github.com/shy2116/project1/blob/master/fill%20factor%20and%20fanout.PNG?raw=true)
**Fill factor:** The portion of a page that is initially used to store data. The space leftover is a buffer to mitigate potentially expensive insertion costs. Empirical tests show that ~66% is the optimal factor
**Fanout (“branching factor”):** How many pointers a directory page contains; how many children a given node can have.
**Height:** Length of the path from the root to the leaf node. If a “Secondary” Index Structure is used, the page storing the data is not part of the height.

### Some Numbers (8kb pages)
![](https://github.com/shy2116/project1/blob/master/Some%20numbers.PNG?raw=true)
- We can find the other data entries easily by using only the data in height 2 and height 3 data, therefore, we only store a little data in memory but visit a huge quantity of data easily.
If we use 8kb pages to store integers and pointers, we can store roughly 500 entries/page. At fill factor of 66%, this is roughly 300 entries/page. We can see that the number of entries we can store ramp up exponentially with the height of the tree. A tree of height 2 filled to its limit (66% fill factor) of 27 million integer/pointer entries takes up ~2.4MB of space. Considering laptops today come standard with at least 4GB of memory, 2.4MB can be stored entirely in memory, meaning no disk access is necessary. This is even possible with a tree of height 3, requiring 750MB. 

### Hash Index
![](https://github.com/shy2116/project1/blob/master/Hash%20Index.PNG?raw=true)

- A hash index is a collection of buckets organized in an array. A hash function is used to map search keys to corresponding buckets. A hash function is useful, as it maps data of arbitrary size to data of fixed size.
- Can only support equality predicates. Hash functions are supposed to be random, so it cannot support range operations.
- Can hash on one or more attributes
- Caution: If I use a poor hash function or the data is skewed, it may degenerate to a linked list data structure, because there will be more possibility to meet the collision when insert a page and will induce the reduce of performance.

### Recap
**Question:** How can we access data quickly?

**Thought:** We need different options with different trade-offs to compare.
- **Options: Datastructure** 
 1. Heap file
 2. Directory on top of heap file
 3. Sorted heap file
- **Options: Index Structure**
 1. Primary
 2. Secondary

### Cost
- How you can analyze and decide the cost for each of these things?
**Assumptions**
- Heap File: equality on a key
- Sorted File: files compacted after deletion
- B+ Tree: 100 pages/directory page, 80% fill factor
- Hash Index: no overflow, 80% fill factor

![](https://github.com/shy2116/project1/blob/master/Cost%20table.PNG?raw=true)

**B:** # of data pages
**D:** time to read/write a page
**M:** # of pages in a range query

- **Highlights:**
 - Heap file: the cost of doing a scan
 - B+ Tree: Fill factor affects the number of leaf pages, as well as how many entries a directory page can contain 
 - Hash table: you need to initially make some generous assumptions

#How do you pick?
- Depend on the query
 - Workload: distribution of all the queries expected to be run in the future
   - Based on past access patterns, what do we expect future access patterns to look like?
 - Relations
 - Attributes
 - Types of predicates (=, <, >)
 - Selectivity
   - Insert/Delete/Update 
   - Quantity
- **Example 1:** If running solely inserts, then use a heap file
- **Example 2:** If we have a uniform data distribution, only need equality constraints, and have no need for inserts: hash index

**General guideline: Everything is predicated on the query you are running and its WHERE clause. Put a different way, the attributes and the types of predicates being applied to those attributes.**
