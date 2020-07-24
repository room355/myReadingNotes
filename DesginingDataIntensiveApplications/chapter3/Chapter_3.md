# Chapter 3 - Storage and Retrieval

##### About this chapter
Database: 
+ store the data
+ read the data

Need to select storage engine accordingly for the application, and we will cover these two store engine in the following:
+ *log-structured* storage
+ *page-oriented* storage

##### Data Structures That Power You Database

Many databases internally use a *log*, (*an append-only data file* and it is efficient)

Databases' important issues:
+ concurrency control
+ reclaiming disk space (log can't gorw forever)
+ handling errors
+ partially written records

In order to efficiently lookup a database, **index** is needed. 
An index is an additional structure derived from the primary data.
> Don't affect contents of database, 
> but impacts the performance of queires. (why)
> Because maintaing additional structures incurs overhead, especially on writes.

Important trade-off in storage systems:
+ well-choosen indexes speed up read queries
+ every index slows down writes

##### Hash Indexes
*indexs* for *key-value* data. 
1. commonly used (similar to *dictionary* type)
2. usually implemented as a hashmap

Reminder: why don't we use hash map to index our data on disk?
One implementation (indexing strategy) is:
+ keep an in-memory hashmap
    + *Key* is mapped to a byte offset in the data file (location at which *value* can be found)
    > When writing (append a new k-v pair), also need to update the hash map to reflect the offset of the writing data (both for **insert** and **update**)
    > When reading (look up a value), use the hashmap to find the offset and read the value
![image](https://drive.google.com/uc?export=view&id=1E-hCKwwfZbKeoj-4D5C8QJCCpSQzbm2M)
Bitcask use this approach for index.

**Question:**
How to avoid disk space running out since we only keep appending to a file?
**Solution**:
1. Break the log into segments of certain size, making subsequent writes to a new segment when the size is reached.
2. Perform *compaction* 
    > Throwing duplicate keys and keeping only the most recent update for each key. (Similar behavior like [LRU Cache](https://leetcode.com/problems/lru-cache/) but they are two different things)
3. Merge several *segements* together at same time performing the *compaction*.
![image](https://drive.google.com/uc?export=view&id=1ci1J3-mbtVW1iDYhyogF1iKSfBEAPelu)
Merge and compaction of frozen segements can be done in a background thread. At the meanwhile, we can still use the old segments continue on reading and writing. Reading continue on old segment, once the merge is done, read switch to the new segment.

Works in detail:
> Each segment now has its own in-memory hash table, mapping keys to file offsets. In order to find the value for a key, we first check the most recent segment’s hash map; if the key is not present we check the second-most-recent segment, and so on. The merging process keeps the number of segments small, so lookups don’t need to check many hash maps.

Note: Leads to a question , how do we guarantee thread safe in this scenario?

**Related Post:**
[A Blog on segment in HBASE](https://blogs.apache.org/hbase/entry/accordion-developer-view-of-in)
In short: Handled by pipeline (copy a clone of the pipeline and protected by lock), reorder the segment, maintain a snapshot, keep tracking of segment version, etc.

**Important Issues**
+ File Format
    + Faster and simpler to use a binary formatr that first encodes the length of a string in bytes, followed by the raw string. (Maybe encoding like Run Length Encoding?)
+ Deleting records
    + Need to append a special deletion record to the data (*tombstone*)
        > When log segments are merged, the tombstone tells the merging process to discard any previous values for the deleted key.
+ Crash recovery
    + in-memo hash maps are lost, so can restore each sgement's hash map by reading entire segment file and noting the offset for every key.
        > It will have performance issue when data volume is large, can speed up by using snapshot (storing snapshot of each segment's hash map on disk)
+ Patially written records
    + Protect the database from corrupted data by using checksum to detect the corrupted data.
+ Concurrency control
    + Only one write thread. Data file segments are append-only and otherwise immutable, so they can be read concurrently by multiple threads. 


**Why we use append-only design? Why not updating the old value?**
+ Appending is much faster than random write.
+ Concurrency and crash recovery are much simpler if files are append-only or immutable.
+ Merging old segments avoids the problem of data files getting fragmented.

**Advantage:**
+ Well suited to each key is updated frequently
+ High performance for read/write
+ The values can use more space than there is available memory since they can be loaded from disk with just one disk seek
    > My assumption is we use hash idnex and we can get the offset at one time, then we can load the data at once.

**Limitations**:
+ The hash table must fit in memory. Hash map has to stored in memory since it has bad performance when stored in disk. 
+ Range queries are not efficient. O(N) scan for range.

##### SSTables and LSM-Trees
**Sorted String Table (SSTable):** 
Requirement:
+ Requiring the sequence of k-v pairs is *sorted by key.*
+ Requiring each key only appears once within each merged segment file.

Advantages:
+ Mergging segements is simple and efficient.
    > Start reading the input files side by side, look at the first key in each file, copy the lowest key (according to the sort order) to the output file, and repeat. This produces a new merged segment file, also sorted by key. 
    
    > Q: What if same key appears in serval input segments?
      A: When multiple segments contain the same key, we can keep the value from the most recent segment and discard the values in older segments.
      Since all value are stored during the time range and thus the most recent one will be valid.
    
    > The whole process works like below:
    ![image](https://drive.google.com/uc?export=view&id=1T4zJaRPOLBlOZBasz8pF7D_9hrTqvoQo)
    Note: Works similar like [Merge K sorted list](https://leetcode.com/problems/merge-k-sorted-lists/), but will replace the value with the most recent one
+ No need for keep an index for all the key in memory.
    > Key is sorted so we can jump to the offset directly at some point and keep on searching.
    > Still need an in-memory index to tell you the offsets for some of the keys, but it can be sparse
    ![image](https://drive.google.com/uc?export=view&id=12LYddfyKfoTU0qf8Gf5ug2bQqLpDPSVk)
+ It is possible to group records into a block and compress it before writing it to disk.
    > Each entry of the sparse in-memory index then points at the start of a compressed block. 
    > Saving disk space, compression also reduces the I/O bandwidth use.
    
##### Constructing and maintaing SSTables
Make the Storage engine work as follows:
+ Add a write to an in-memory balanced tree data structure *(memtable)*
+ Write *memtable* to disk as an SSTable file when it reach a certain size. And this new SSTable becomes the most segment of the database. New writes will continue on a new memtable while SSTable being saved to the disk.
+ For reading, first check the *memtable* then the most recent segment, and the next older segment, etc.
+ Run a merge and compaxtion process in the background to combine segment files. Discard overwritten or deleted values.

> Q: How to we handle database crash and lost the most recent write?
> A: Keep an log on disk to append every write record. And discard the log when the memtable is written out to an SSTable. It just helps on restoring memtable after a crash.

##### Making an LSM-tree out of SSTables
**Log-Structured Merge-Tree (LSM-Tree)** storage engines: Storage engines that are based on the aboved principle of merging and compacting sorted files.

##### Performance optimizations
The LSM-tree algorithm can be slow in some scenario. 
For example, need to check all the way from top to bottom if a key is non-existed.
+ Optimize the access to a key: *Bloom filters* (Cassandra use this, In short:)
    + If key doesn't exist, it will tells you a confirmative 'NO' (hashed index will be 0)
    + But if a key may exist, means some of the hashed index will be 1
+ Different strategies to determine the order and timing of how SSTables are compacted and merged.
    + size-tiered compaction 
        + Newer and smaller SSTables are successively merged into older and larger SSTables. 
    + leveled compaction
        + Key range is split up into smaller SSTables and older data is moved into separate "levels"

    [How Cassandra think on these two strategies](https://www.datastax.com/blog/2011/10/leveled-compaction-apache-cassandra#:~:text=Leveled%20compaction%20guarantees%20that%2090,be%20wasted%20by%20obsolete%20rows.)
The basic idea of LSM-Tree: Keeping a cascade of SSTables that are merged in the background.
 
##### B-Tree
+ Like SSTables, B-tree keep key-value pairs sorted by key
+ B-trees break the database down into fixed-size *blocks* or *pages*

Each *page* can be identified using an address or location, which allows one page refer to another. Thus we can use page reference to construct a tree of pages like below:
![image](https://drive.google.com/uc?export=view&id=1lxx8_5VHFb-_ggMW4Q92Z0RISXy9uZ4V)

Steps:
1. One page is designated as the *root* of the B-tree (starting point of traverse)
2. The page contains several keys and references to child pages.
    + keys between the reference indicate the boundaries.
3. Eventually will traverse to a page containing individual keys(*a leaf page*)

The number of references to child pages: *branching factor*

**When update:**
1. Search for the leaf page containing that key
2. Change the value in that page
3. Write the page back to disk (won't affect other reference)

**When adding new key:**
1. Find the page with range encompasses the new key and add it
    > If there isn't enough space in the page,
    It will split into two half-full pages and update the parent page

The B-tree algorithm ensures that the tree remain *balanced*. 
A B-tree with n keys always has a depth of O(log *n*)

##### Making B-tree reliable
**Adding a new key** is a dangerous operation.
If database crashes after only some of the page have been written, index will be corrupted.

**In order to handle this:**
A *write-ahead log* *(A.K.A WAL, redo log)* is needed.
An append-only log to which every B-tree modification must be written before it can be applied to the pages of the tree itself.
Can use this log to restore the database.

**Another problem** is concurreny:
Pages in page becomes inconsistent when multiple thread accessing the B-tree at the same time.
Can be protected by using *latches* (light-weight locks) (key range lock)

##### B-tree optimizations
1. Use a copy-on-write scheme (like snapshot) for modified pages instead of using *WAL*. (Will be useful for concurrency control)
2. Abbreviate keys. Packing more keys into a page allows the tree to have a higher branch fator, and fewer levels.
3. Try to lay out the tree so that leaf pages appear in sequential order in disk. (To save the effort when a disk seek is required for revery page that is read)
4. Additional pointers to jump to different pages(for example, sibling pages)

##### Comparing B-Trees and LSM-Trees
Reads are typicllay slower on LSM-trees
+ they have to check different data structures and SSTables have different stages of compaction.

**LSM-Tree Advantage**
*Write amplification:* one write to the database resulting in multiple writes to the disk over the course of the database’s lifetime.
> Extra write is needed when databases crashes protection is needed, or B-Tree need to split current page and overwrite parent page, LSM-Tree's compaction, etc.

Advantages:
1. Normally, LSM-trees have lower write amplification. Since:
    + LSM-tree sequentially write compact SSTable files rather than having to overwrite several pages in the tree.
2. LSM-trees can be compressed better, have a more efficient storge strategy.
    + B-tree leave some disk space unused due to [fragmentation](https://en.wikipedia.org/wiki/Fragmentation_(computing)#:~:text=of%20size%200x4000.-,Data%20fragmentation,that%20are%20not%20close%20together.&text=When%20a%20new%20file%20is,fit%20into%20the%20available%20holes.).

**LSM-Tree Downsides**
1. The compaction process can sometimes interfere with the performance of ongoing reads and writes. (average response time is fine, but at higher percentiles the response time can be quite slow)
    + Due to disk have limited resources 

2. If write throughput is high and compaction is not configured carefully, it can happen that compaction can't keep up with the rate of incoming writes.
    + Leads to unmerged segments keep growing till run out of disk space.
3. An advantage of B-tree is each key exists exactly one place in the index, but log-structed storage engine may have multiple copies of the same key in different segments.
    + This makes B-tree easier to provide support for transactional, since it is normally implemented using locks on ranges of keys.

##### Other Indexing Structures
It is very common to have *secondary indexes*, and there might be many rows with the same key, and this can be solved in two ways:
+ Making each value in the index a list of matching row identifiers 
+ making each key unique by appending a row identifier to it

**Storing values within the index**
1. The value can be reference to the actual data and the data is stored in *heap file*.
Will be efficient when updating value since we can directly update the value,
But if the new value is larger then it needs to moved to a new location in *heap* with enough space. (*nonclustered index*)
    + If that's the case, might need to update all indexes and point at the new heap location of the data, or a forwarding pointer is left behind in the old heap location.
2. If the penalty of reading when the cost for accessing from index to heap is too hugh, it is better to sotre the indexed row directly within an index. (*clustered index*)
    + A clustered index is a special type of index that reorders the way records in the table are physically stored.
    + [further explaination on clustered index](https://stackoverflow.com/questions/1251636/what-do-clustered-and-non-clustered-index-actually-mean#:~:text=A%20clustered%20index%20is%20a,index%20contain%20the%20data%20pages.)
3. A compromise between a clustered index and nonclustered index is: *covering index* (or *index with included columns*) - stores some of a table's columns within the index. 
    + the index is said to *cover* the query (An index that contains all information required to resolve the query is known as a “Covering Index”)
    + [further explaination on covering index](https://stackoverflow.com/questions/609343/what-are-covering-indexes-and-covered-queries-in-sql-server)

Clustered and covering indexes can speed up reads but require additional storage and can add overhead on writes.

**Multi-column indexes**
1. *concatenated index*
    + Combines several fields into one key by appending one column to another
2. *Multi-dimensional indexes*
    + Translate a two-dimensional location into a single number

**Full-text search and fuzzy indexes**
Allow to search for *similar* keys
For example, A FSM (Finite State Machine) efficient search for words within a given edit distance.

**Keeping everything in memory**
Basically it is promosing if we can use a in-memory database.
> they can be faster because they can avoid the overheads of encoding in-memory data structures in a form that can be written to disk 

##### Transaction Processing or Analytics?
*Transaction processing* just means allowing clients to make low-latency reads and writes (ACID not necessarily)
+ *online transaction processing* (OLTP)
    + applications that are interactive (user have access to CRUD)
    + small number of records per query, fetched by key
+ *data analytics*
    + hugh amount of records
    + aggregation statistics
    + read only few column in record

A separate database in needed to run the analytics: *data warehouse*.

**Data Warehousing**
Expectation on OLTP systems:
+ highly available and to process trans‐ actions with low latency

ad hoc queries on an OLPT database will be expensive, scanning large parts of the dataset, which can harm the performance of concurrently executing transactions.

A data warehouse, by contrast:
1. Contains read-only copy of data
2. Data is extracted from OLTP databases
3. Data tranformed into an analysis-friendly schema, cleaned up, loaded into data warehouse.

This process is called *Extract-Transform-Load* (ETL)
![image](https://drive.google.com/uc?export=view&id=1AH-ZADZyOyVt6dMHFedB4QU_0gsJDHvg)

The data warehouse can be optimized for analytic access patterns.
So even both *OLTP* and *data warehouse* sharing same sort of SQL interface, but they are optimized for different access patterns.

Data warehouse models:
+ *star schema*
    + the center of the schema is *fact table* 
    + Other columns in the face table are foreign key references to other tables, called *dimension tables*
+ *snowflake schema*
    + dimensions are further broken down into subdimensions

##### Column-Oriented Storage
In most OLTP databases, storage is laid out in a *row-oriented fashion*:
+ all the values from one row of a table are stored next to each other.

*column-oriented storage*:
+ don’t store all the values from one row together, but store all the values from each column together instead.
+ The column-oriented storage layout relies on each column file containing the rows in the same order. 

##### Column Compression
+ *bitmap encoding*
    > take a column with n distinct values and turn it into n separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has that value, and 0 if not.

The bottleneck for data warehouse queries is the bandwidth for getting data from disk into memory.
> Column compression allows more rows from a column to fit in the same amount of cache. Operators, such as the bitwise AND and OR can be designed to operate on such chunks of compressed column data directly. This is known as **vectorized processing**

##### Sort Order in Column Storage
Another Advantage of sorted order: help with column compression
row-oriented storage:
+ Keeps every row in one place
+ secondary indexes just contain pointers to the matching rows

##### Writing to Column-Oriented Storage
Most of load consist of large read-only queries and the optimization above making writes more difficult.
One solution is: 
1. LSM-trees. All writes first go to an in-memory store, where they are added to a sorted structure and prepared for writing to disk. It doesn’t matter whether the in-memory store is row-oriented or column-oriented. When enough writes have accumulated, they are merged with the column files on disk and written to new files in bulk. 
2. Queries need to examine both the column data on disk and the recent writes in mem‐ ory, and combine the two. 

##### Aggregation
*materialized aggregates*:
It is wasteful to crunch the raw data every time, can be optimized by cache.
+ *materialized view*:
    + An actual copy of the query result
    + Makes write expensive
+ *virtual view*:
    + A shortcut for writing queries
+ Advantage of *mv*:
    + Certain queries become very fast since they have been precomputed
+ Disadvantage of *mv*:
    +  Lost the flexibility as querying raw data