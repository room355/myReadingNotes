# Chapter 5 - Replication
*Replication* - Keeping a copy of the same data on multiple machines that are connected via a network.

Reasons for replication:
+ **Reduce latency** 
    + Keep data geographically close to your users 
+ **Increase availability**
    + Allow system keep on working even some of its part failed 
+ **Increase read throughput**
    + Scale out the numbers of nodes to serve read queries 

*Replication Algorithm*
+ *single-leader*
+ *multi-leader*
+ *leaderless*

### Leaders and Followers
*replica* - A copy of the database sotred in node. 

*leader-based replication (active/passive or master-slave replication)*
1. *leader (master or primary)*
    + Clients' write requests was sent to leader first, and the leader will write the new data to its local storage
2. *followers (read replicas, slaves, secondaries, or hot standbys)*
    + The leader sends the data change to all its followers as part of *replication log* or *change stream*
    + Each follower takes the log from the leader and update its local copy of the database
3. *client*
    + When client sends read request, it can query from either the leader or any one of the followers.
    + When client send write request, it will only be accepted by the leader

**Synchronous vs Asynronous Replication**
*synchronous* - the leader will wait unit the follower confirmed it received the write before reporting success to the user.
*asynchronous* - the leader sends the message but doesn't wait for a response from follower
![image](https://drive.google.com/uc?export=view&id=1WPwaAzXrs24RM4ZCiGkaxsOrKJNbJE0K)

+ Advantage of synchronous replication:
    + The follower is guaranteed to have an up-to-date copy of the data (consistent with the leader)
+ Disadvantage of synchronous replication:
    + The write can't process without the follower to respond and the leader will block all the write

Thus we can have *semi-synchronous*:
Have an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower.

+ Advantage of asynchronous replication:
    + Leader can continue processing writes even all of its followers have fallen behind. 
+ Disadvantage of asynchronous replication:
    + Durability weaken
    
+ [Chain replication](http://muratbuffalo.blogspot.com/2011/02/chain-replication-for-supporting-high.html#:~:text=Chain%20replication%20(2004)%20is%20a,retake%20of%20primary%2Dbackup%20replication.)
    + Read, Write handled in a sequence fashion, the implementation is like linkedlist
    + A *master* process will handle node failure
        + When *head* node failed, *master* will remove failed *head* node and replace it with a successor *head* node
        + When *the node in the middle failed*, it will remove that node and connect its *previous* node and *next* node

**Setting up New Followers**
How to ensure that the new follower has an accurate copy of the leader's data?
1. Take a consistent snapshot of the leader's database at some point.
2. Copy the snapshot to the new follower node.
3. The follower connects to the leader and requests all the data changes that have happened since the snapshot was taken.
    + This requires that the snapshot is associated with an exact position in the leader’s replication log. 
4. When the follower has processed the backlog of data changes - *caught up*

**Handling Node Outage**
Goal: Being able to reboot individual nodes without downtime
+ Follower failure: Catch-up recovery
    + Followers keeps a log of the data changes received from the leader.
+ Leader failure: *failover* 
    + *failover*: one of the followers needs to be promoted to be the new leader, clients need to be reconfigured to send their writes to the new leader, and the other followers need to start consuming data changes from the new leader. 
        + Automatic failover
            1. Determining that the leader has failed. 
                + Measuring timeout is one of the method 
            2. Choosing a new leader.
                + Done by an election process, could be appointed by a previously elected *controller node*. 
                + The best candidate have the most up-to-date data changes replica
            3. Reconfiguring the system to use the new leader.
                + Make sure old leader becomes a follower and recognize the new leader

What might go wrong:
+ If asynchronous replication is used, the new leader may not have received all the writes from the old leader before it failed. 
    + Solution: The old leader’s unreplicated writes to simply be discarded, which may violate clients’ durability expectations.
    + Discarding write is dangerous if other storage systems outside of the database need to be coordinated with the database content.
+ *split brain* - Two nodes both believe they are the leader. 
    + Data might be lost or corrupted
    + Solution: shut down one node when two leaders are detected
+ Incorrect timeout choice to determine the leader is down. 

##### Implementation of Replication Logs
**Statement-based Replication**
Implementation: 
+ Logs every write request *(statement)* that it executes and sends that statement log to its follower

Advantage:
+ Great Readability on CQL operation
Disadvantage:
+ Will have problem if a statement calls a nondeterministic function
+ Will have problem if statement depend on the existing data *(e.g., Update ... WHERE ...)*
+ Statements with side effects *(e.g., triggers, stored procedures, user-defined functions)*

**Write-ahead log(WAL) shipping**
Implementation: 
+ The log is an append-only sequence of bytes containing all writes to the database. 

Advantage:
+ Can build a replica on different node base on the same log.

Disadvantage:
+ The log describe the data on a very low level
    + Make replication closly coupled to the storage engine. Will have problem if engine version is different.

**Logical (row-based) log replication**
Implementation: 
+ A *logical log* for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row:
    + For an inserted row, the log contains the new values of all columns.
    + For a deleted row, the log contains enough information to uniquely identify the row that was deleted. Typically this would be the primary key, but if there is no primary key on the table, the old values of all columns need to be logged.
    + For an updated row, the log contains enough information to uniquely identify the updated row, and the new values of all columns (or at least the new values of all columns that changed).

Advantage:
+ Logical log is decoupled from the storage engine internals.
+ Easier for external applications to parse. (*change data capture*)

Won't discuss disadvantage in this post, since the disadvantage of logical log replication is really depend on the way of database design and implement it.

**Trigger-based replication**
Implementation: 
+ *triggers* and *stored procedures*
    + A trigger lets you register custom application code that is automatically executed when a data change (write transaction) occurs in a database system. 

Advantage:
+ Great flexibility

Disadvantage:
+ Trigger-based replication typically has greater overheads than other replication methods, and is more prone to bugs and limitations than the database’s built-in replication.

### Problems with Replication Lag
Apart from failure tolerance, replication alsoneed to notice:
+ Scalability
+ Latency

*scenario:*
For workloads that consist of mostly reads and only a small percentage of writes (a common pattern on the web)

*An good option is:*
Create many followers, and distribute the read requests across those followers. 
+ This remove load from the leader and allows read requests to be served by nearby replicas.

Above acrchitecture is called *read-scaling*:
Advantage of *read-scaling*:
+ Increase the capacity for serving read-only requests simply by adding more followers.

Problems:
+ Only works with asynchronous replication, 
    + Since in synchronous replication, a single node failure would make the whole system unavailable for writing.
+ Will have inconsistence, if some nodes has fallen behind, 
    + run the same query on the leader and a follower at the same time, you may get different results, because not all writes have been reflected in the follower.

**eventual consistency and the replication lag**
*eventual consistency* 
+ The incosistence is temporary, and the data change will be *catught up*

*The replication lag*
+ The delay between a write happening on the leader and being reflected on a follower.

##### Reading Your Own Writes
*Scenario:*
When new data is submitted, it must be sent to the leader, but when the user views the data, it can be read from a follower. 
In asyn-replication, the new data may not yet have reached the replica. 
Example like below:
![image](https://drive.google.com/uc?export=view&id=1TJ1u1N1JpNG2Ix4VY18Yu3pwPHcWqbcK)

*Goal:*
Need *read-after-write consistency*, a.k.a *read-your-writes consistency*

*Solution:*
+ When reading modified data, read it from the leader; otherwise, read it from a follower.
    + How do we know that data is modified recently?
        + A naive solution is some data will be modified alot so we will only read it from the leader.  
        + Track the time of the last update and make all reads from the leader within some time range.
        + The client can remember the timestamp of its most recent write—then the system can ensure that the replica serving any reads for that user reflects updates at least until that timestamp.
            + The timestamp could be a *logical timestamp* 
        + If replica distributed across multiple datacenters
            + Any request that needs to be served by the leader must be routed to the datacenter that contains the leader. 

*Potential issues on the solution:*
+ *cross-device* read-after-write consistency
    + Approaches that require remembering the timestamp of the user’s last update become more difficult, because the code running on one device doesn’t know what updates have happened on the other device. This metadata will need to be centralized.
    + If your replicas are distributed across different datacenters, there is no guarantee that connections from different devices will be routed to the same datacenter. 

##### Monotonic Reads
*Scenario:*
It is possible user see thing moving backward in time.
+ A user make serval reads from different replicas and the lagging follower hasn't picked up that write.

Example like below:
![image](https://drive.google.com/uc?export=view&id=1_LM275tgMrFVjjolJbsPz8uq_gKeNY9a)

*Goal:*
+ Will not read older data after having previously read newer data.

*Solution:*
+ Make sure that each user always makes their read from the same replica
    + For example, the replica can be chosen based on a hash of the user ID

##### Consistent Prefix Reads
violation of causality: 
+ A obersever might see the data misorder for lagging replication.
Example like below:
![image](https://drive.google.com/uc?export=view&id=1EEHND3BDQ12eyYfQ6jh9V2V5zjp8ivVw)

*Goal:*
+ If a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.

*Solution:*
+ *Consistent prefix reads*
    + Make sure that any writes that are causally related to each other are written to the same partition.

### Multi-Leader Replication
Problem with Single-Leader replication:
You can't write if the leader is disconnected.

Solution:
*multi-leader configuration*
+ Allow more than one node to accept writes.
+ Each node that processes a write must forward that data change to all the other

##### Use Cases for Multi-Leader Replication
In a multi-leader configuration, you can have a leader in each datacenter.

**Comparsion of Single-Leader vs Multi-Leader**
+ *Performance*
    + Single-Leader
        + Every write must go over the internet to the datacenter with the leader. This can add significant latency to writes and might contravene the purpose of having multiple datacenters in the first place. 
    + Multi-Leader
        + Every write can be processed in the local datacenter and is replicated asynchronously to the other datacenters. Thus better latency.
+ *Tolerance of datacenter outages* 
    + Single-Leader
        + failover needs to be carried out
    + Multi-Leader
        + Continue on operating independently and replication catches up when the failed datacenter comes back online.
+ *Tolerance of network problem*
    + Single-Leader
        + Sensitive to inter-datacenter link
    + Multi-Leader
        + With asynchronous replication, a temp network issue won't interrupt write process
 
**Down sides of multi-leader:**
+ write conflicts might happens and needs to be resolved. 
    + e.g., the same data may be concurrently modified in two different datacenters
+ There are often subtle configuration pitfalls and surprising interactions with other database features. 
    + e.g., autoincrementing keys, triggers, and integrity constraints can be problematic.

**Some good scenario works well with multi-leader:**
+ Clients with offline operation
    + Multi-Leader is appropriate if you have an application that needs to continue to work while it is disconnected from the internet.
        + In this case, every device has a local database that acts as a leader (it accepts write requests), and there is an asynchronous multi-leader replication process (sync) between the replicas of your calendar on all of your devices. The replication lag may be hours or even days, depending on when you have internet access available.
+ Collaborative editing
    + Allows serval writes edit a document simulaneously.
        + Changes applied to their local replica and asy-replicated to the server

##### Handling Write Conflicts
Problem with multi-leader:
+ Write conflict can occur
    + Example like below:
    ![image](https://drive.google.com/uc?export=view&id=1PsKkqTNoaRfwFvDTprOTzyZcq9gX-jc6)

**Syn vs Asyn conflict detection:**
+ Single-Leader
    + Second write will be blocked until first write complete
+ Multi-Leader
    + Both write will be successful is only detected asynchronously later
        + Could make conflict detectyion syn but will lose the advantage of multi-leader:
            + Allow each replica to accept writes independently

**Conflict avoidance**
*Write conflict solution - Just avoid conflict:*
+ If the application can ensure that all writes for a particular record go through the same leader, then conflicts cannot occur
    + For example, when a user edit their own data:
        + Can ensure that requests from a particular user are always routed to the same datacenter and use the leader in that datacenter for reading and writing. 
+ But,
    + If change the designated leader
        + Conflict avoidance will break down, since concurrent writes may happen

**Converging toward a consistent state**
+ Single-Leader
    + *LWW* (last write wins)
+ Multi-Leader
    + If *LWW* applied,
        + Data will be inconsistent
        + The database must resolve the conflict in a *convergent* way, 
            + All replicas must arrive at the same final value when all changes have been replicated 
    + To achieve that,
        + Give each write a unique ID and applied *LWW* (highest ID wins for example) - may cause data loss
        + Give each replica a unique ID, higher ID higher precedance - may cause data loss
    + Merge the values together
    + Reocrd the conflict in an explicit data structure that preserves all information with corresponding conflict-resolving code

**Custom conflict resolution logic**
Conflict-resolving application can be implemented on *write* or on *read*:
+ *On write*
    + Conflict handler is called whenever conflict is detected
        + Handler typically cannot prompt a user-it runs in a background and must execute quickly 
+ *On read*
    + Multiple versions of conflicting writes are stored and returned to the application
        + May prompt the user or automatically resolve the conflict and write the result back

**What is a conflict?**
+ Obvious conflict 
    + Same field same record different values
+ Subtle conflict 
    + Different records inserted at the same time to the same entry 

##### Multi-Leader Replication Topologies
*replication topology*: 
+ Describes the communication paths along which writes are propagated from one node to another.

![image](https://drive.google.com/uc?export=view&id=1rqjvixiS4aR0kaf3-w6Ieg3jhNZfOQ1-)

+ *all-to-all*
    + Every leader sends its writes to every other leader 
    + Advantage over *circular* and *star*
        + It allows messages to travel along different paths, avoiding a single point of failure.
    + Problems:
        + Some replication messages may overtake others due to network lag
        + Example:
        ![image](https://drive.google.com/uc?export=view&id=1OzobiCxukl75FP5-4NCsgjEXtAMEEN9X)
        + To solve this:
            + Make sure insertion first, then update
                + Timestamp is not sufficient since clocks cannot be trusted
                + *version vectors* is needed
+ *circular* and *star*
    + *circular* 
        + Each node receives writes from one node and forwards writes to one other node 
    + *star*
        + A root node forwards writes to all of the other nodes 
    + Problems:
        + Dependency on one or many nodes 
            + A write may need to pass thru several nodes before it reaches all replicas 
                + To prevent this, identifiers are needed (write is tagged with all the id of nodes it pass thru)
            + If one node fails, it can interrupt flow of replications

### Leaderless Replication
*Dynamo-style*: Allowing any replica to directly accept writes from clients
A coordinator may exists to enfore a particular order of writes

##### Writes to the Database When a Node Is Down
*failover* will not be needed, since if any of the node received the write.

May get *stale* (outdated) values when the node comes back online. To solve this:
+ *Read requests are also sent to several nodes in parallel.*

##### Read repair and anti-entropy
How an unavailable node comes back online can catch up on the writes that is missed?

+ *Read repair*
    + When a client makes a read from several nodes in parallel it can detect any stale responses and writes newer value back to that node. (works fine when read is frequent)
    + Only performed when read happened to a value.
+ *Anti-entropy process*
    + Constantly looks for differences in the data between replicas and copies any missing data from one replica to another.
    (does not copy in any particular order)

##### Quorums for reading and writing
If there are *n* replica, writer must be confirmed by *w* nodes (successful writes), must query *r* nodes for each read.
Need to satisfy:
+ *w* + *r* > *n*

Reads and writes that obey these *r* and *w* are called *quorum* reads and writes. (It ensure at least one of the *r* nodes must be up to date)

In dynamo-style db:
+ *n*, *w* and *r* are configurable 
+ *n* is choosen as an odd number
+ set *w* = *r* = (*n* + 1) / 2
    + If *r* is too small, it can make read faster but just one failed node causes all db writes to fail

The quorum condition, allows the system to tolerate unavailable nodes as follows:
+ If w < n, we can still process writes if a node is unavailable.
+ If r < n, we can still process reads if a node is unavailable.
+ With n = 3, w = 2, r = 2 we can tolerate one unavailable node.
+ With n = 5, w = 3, r = 3 we can tolerate two unavailable nodes.
    > ![image](https://drive.google.com/uc?export=view&id=1Mez-mIRMDq2U7KJvdOhoMg62LGRmGskQ)
+ Normally, reads and writes are always sent to all n replicas in parallel. 
+ The parameters *w* and *r* determine how many nodes we wait for—i.e., how many of the *n* nodes need to report success before we consider the read or write to be successful.

##### Limitations of Quorum Consistency
Often, *r* and *w* are chosen >= n / 2, since *w* + *r* > *n* while still tolerating up to *n*/2 node failures.
The set of nodes that have written data must overlap with the set of nodes that user reads.

It is possible to set *w* and *r* smaller, 
+ cons: more likely your read didn't include the node with the latest value.
+ pros: lower latency and higher availability

Edge cases when stale values are returned:
+ *sloppy quorum* may cause *w* writes may end up on different nodes than the *r* reads, no guarante on *w* nodes and *r* nodes are overlapping.
+ Concurrent write
+ Write concurrently with read
+ write succeeded on some replicas but failed on others, and overall succeeded on fewer than *w* replicas and the write is not rolling back. Read may return incorrect value for that write (which is consider as failed)
+ If a node carrying new value fails and its data restored from a replica carrying old value, the number of replicas storing the new value may fall below *w*, breaking the quorum condition.

##### Sloppy Quorums and Hinted Handoff
If the client can connect to some database nodes during the network interruption, not up to the nodes that needs to satisify the quorum condition. In this case, will face a trade-off:
+ Is it better to return errors to all requests for which we cannot reach a quorum of *w* or *r* nodes?
+ Or should we accept writes anyway, and write them to some nodes that are reachable but aren’t among the *n* nodes on which the value usually lives? (*sloppy quorum*)

*sloppy quorum*:
+ Writes and reads still require *w* and *r* successful responses, but those may include nodes that are not among the designated *n* “home” nodes for a value.

+ Pros: Increase write availability
+ Cons: Cannot be sure to read the latest value for a key, because the latest value may have been temporarily written to some nodes outside of *n*

*hinted handoff*:
+ Once the network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate “home” nodes.

Thus, no guarantee that a read of r nodes will see it until the hinted handoff has completed.

###### Detecting Concurrent Writes
Dyamo-style databases allow several clients to concurrently write to same key. Thus write conflict will occur.
Thus the replica should converge toward the same value. How?

**LWW (discarding concurrent writes)**
The writes are *concurrent*, so their order is undefined.
Can enforce an arbitrary order on writes.
+ Attach a timestamp to each writes, pick the latest timestamp as the most 'recent' and discard any writes with an eariler timestamp.

The cost of *LWW*:
+ Durability: only one of the writes survive and the others will be silently discarded if the client send multiple writes with successful response.
+ With caching, lost writes may be acceptable for *LWW*. But *LWW* is not a good choice if losing data isn't allowed.

*LWW* will be safe if:
Ensure that a key is only written once and thereafter treated as immutable.

##### The "happens-before" relationship and concurrency
How do we decide whether two writes are concurrent?
+ Two operations are *concurrent* if neither happens before the other (neither knows about the other)

##### Capturing the happens-before relationship
Can user version number to track concurrent write on the same key, example like below:
![image](https://drive.google.com/uc?export=view&id=1mtcCk3XZxoRgBf7FuCdvwBYIg3Db3Zj6)
Note: this logic is difficult to explain in few words, please review book's steps.
The server can determine whether two operations are concurrent by looking at the version numbers. And the algorithm works like below:
+ The server maintains a version number for every key, increments the version number every time that key is written, and stores the new version number along with the value written.
+ When a client reads a key, the server returns all values that have not been overwritten, as well as the latest version number. A client must read a key before writing.
+ When a client writes a key, it must include the version number from the prior read, and it must merge together all values that it received in the prior read. (The response from a write request can be like a read, returning all current values, which allows us to chain several writes like in the shopping cart example.)
+ When the server receives a write with a particular version number, it can overwrite all values with that version number or below (since it knows that they have been merged into the new value), but it must keep all values with a higher version number (because those values are concurrent with the incoming write).

##### Merging concurrently written values
Clients must clean up afterward by merging the concurrently written values (*siblings*).
+ *Union* can be used to merge all the concurrent writes
+ When *remove* is required, use *tombstome* as a maker (with appropriate version number) to indicate that the item has been removed.

**Version vectors**
+ The collection of version numbers from all the replicas.

Version vectors are sent from the database replicas to clients when values are read, and need to be sent back to the database when a value is subsequently written. 
