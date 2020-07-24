# Chapter 2
### Relational Model vs Document Model
**Concept of Relational Model:**
Data is organized into relations (*tables* in SQL), where each releation is an unordered collection of **tuples** (*row* in SQL).

**Goal of Relational Model:**
Hide the internal implementation detail behind a clearner interface.

**Related content**
Other models: *network model*, *hierarchical model*

### Birth of NoSQL
**Driving Force**
+ A need for greater scalability than relational databases can easily achieve, including very large datasets or very larger datasets or very high write throughput.
+ A widespread preference for free and open source software over commercial database products.
+ Specialized query operations that are not well supported by the relational model.
+ Frustration with the restrictiveness of relational schemas, and a desire for a more dynamic and expressive data model.

**Related content**
Relational databases will continue to b used alongside a broad variety of nonrelational datastores - *polyglot persistence*[1]
+ [1] [Wiki](https://en.wikipedia.org/wiki/Polyglot_persistence) 
+ [2] [Why Distributed SQL Beats Polyglot Persistence for Building Microservices?](https://blog.yugabyte.com/why-distributed-sql-beats-polyglot-persistence-for-building-microservices/)

### The Object-Relational Mismatch 
**Impedance mismatch**
The disconection between application code and the database model of tables. And this will be handled by Object-relational mapping (ORM).
However, various ways of handling one-to-many relationship are also needed:
+ in the traditional SQL model, the most common normalized representation is to put columns with a foreign reference to join other data model.
+ or add support for structured datatypes (*e.g. JSON, XML*) and support for querying and indexing inside those documents.
+ encoded JSON or XML document and store it in a text column in the db. (Note: lost support for querying for values inside that encoded document).

**Intros**
There are drawback on JSON style data encoding format. "Schema flexibility in the document mode" **p39**

### Many-to-One and Many-to-Many Relationships

Benefits of using id *(e.g. region_id, school_id, etc.)*
+ Consistent style and spelling across profules
+ Avoiding ambiguity
+ Ease of updateing
    >Also, if some information is duplicated, and all the redundant copies need to be updated. That will incurs write overheads, and risks inconsistencies.
    > Removing such duplication is the key idea behind **normalization** in databases.
+ Localization support
+ Btter search
 
Many-to-one releationships doesn't fit well in the document model.
+ lack or weak support for **join**
    > Multiple join are needed for retriving data if join is not well supported.
+ Data has a tendency of becoming more interconnected as features are added to applications.

### Two Models
##### The network model
Conference on Data System Languages (CODASYL) model was a generalization of the hierarchical model.
+ A record can have multiple parents
+ The link between records in the network model were more like pointers.
    > And the only way of accessing a record was to follow a path from a root record was to follow
    > an **access path**: a path from a root record along these chains of links.
+ Like the traversal of a linked list

Advantage: Make the most efficient use of the very limited hardware capablilities in the 1970s.
Disadvantage: Made the code for querying and updating the database complicated and inflexible.

##### The relational model
By Contrast, 
+ Lay out all the data in the open: a relational(table) is simply a collection of tuples(rows).
+ The query optimizer automatically decides which parts of the query to execute in which order, and which indexes to use.
    > Similar like "access path", but the big difference is that they are made automatically by the query optmizer.


**Comparison**
+ difference: document model storing nested records within their parent record rather than in a separate table.
+ similarity: When it comes to representing many-to-one and many-to-many relationships, the related item is referenced by a unique identifier:
    + foreign key
    + document reference

**Related content**
TODO: add links for query optmizer

### Relational VS Document Databases Today
##### Which leads to simpler application code
Main Argument point: 
+ Document data model
    + schema flexibility
    + better performance due to locality
    + closer to the data structures used by the application in some cases
    + Can't refer directly to a nested item within a document                                                                                                                                   
+ Relational model
    + providing better support for joins
    + many-to-one and many-to-many relationships
    + THe relational technique of *shredding* - splitting a document-like structure into multiple tables
        + In some case may lead to cumbersome schemas and unnecessarily complicated application code

Summary:
It really depends on the requirement.
+ If many-to-many relationships is not needed then document model may be better
+ If the application does use many-to-many relationships, *join* can optimize the performance

##### Schema flexibility in the document model

Document databases are called *schemaless*, but it is misleading.
+ *schema-on-read* is more accurate to describe the document model
    + Key: **implicit schema** - The code that reads the data usually assumes some kind of structure.
    + Similar to dynamic (runtime) type checking
    + When an application want to change the format of the data:
        + Tends to start writing with new fields and have code in the application that handles the case when old documents are read
    + Will be advantageous if the items in the collection don't follow same structure, since:
        + There are many different types of objects, and it is not practical to put each type of object in its table
        + The structure of the data is determined by external systems over which you have no control and which may change at any time
+ *schema-on-write* (the traditional approach of relational database)
    + Key: **explicit schema** - The databases ensures all written data conforms to it
    + Similar to static (compile-time) type checking
    + When an application want to change the format of the data:
        + Tends to form a migration with **ALTER** and **UPDATE**, and schema changes have a bad reputation of being slow and requiring downtime
            + **ALTER** should be quick in most relational database (apart from mySQL)
            + **UPDATE** on a large table is likely to be slow due to full table scan and rewrite data.

##### Data locality for queries
*Storage locality* will have advantage when:
+ First, the application often needs to access the entire document.
+ Second, only applies if you need large part of the document at the same time.

So it is generally recommended that to keep documents fairly small and avoid writes that increase the size of a document.
The idea of grouping related data together for locality is not limited to the document model.
+ allowing the schema to declare that a table's rows should be nested within a parent table (Google Spanner)
+ multi-table index cluster tables (Oracle)
+ column-family concept in the bigtable data model (Cassandra and HBase) has a similar purpose of managing locality

### Query Languages for Data
+ declarative query language
    + Just specify the pattern of the data (condition that needs to be met)
    + Hide implementation details of the db engine to introduce performance improvements without changing queries
    + Limited in functionality gives the database more room for automatic optimizations
    + Have a better chance getting faster in parallel execution (only specify the pattern of the result, db is free to use a parallel implementation)
+ imperative code
   + To perform certain operations in a certain order
   + Hard to parallelize across multiple cores and multiple machines (it specifies instructions that must be performed in a particular order)

##### MapReduce Querying
*MapReduce* is:
+ A programming model for processing large amount of data in bulk across many machines.
+ A fairly low-level programming model for distributed execution on a cluster of machines.
+ Its own query language is between declarative and imperative, the logic of the query is expressed with snippets of code. (repeatedly called in framework). It is based on:
    + **map** (a.k.a. **collect**)
    + **reduce** (a.k.a **fold** or **inject**)

Example:
```
// 2. map is called once for every document that matches query
db.observations.mapReduce( function map() {
    var year = this.observationTimestamp.getFullYear();
    var month = this.observationTimestamp.getMonth() + 1; 
    emit(year + "-" + month, this.numAnimals); // 3. The map func emits a key (a string of yy-mm)
},
// 4. the k-v pair emitted by map are group by key, the reduce function is called for all k-v pairs with the same key
function reduce(key, values) { 
    return Array.sum(values); // 5. reduce returns the sum
},

// 1. the filter to consider only shark species can be specified declaratively
{
    query: { family: "Sharks" },
    out: "monthlySharkReport"
} 
);
```
The **map** and **reduce** function must be *pure* function. Means:
+ They must use the data that is passed to them as input
+ They can't perform additional database queries
+ They must not have any side effects (?)

These restrictions allow the db to run the functions any where, in any order and return them on failure. But with powerful logic implementation.

**Usability Problem**
Have to write two carefully coordinated functions. Moreover, a declarative query language offers more opportunities for a query optimizer.
Thus MongoDB added support for a declarative query language called the *aggregation pipeline*.

### Graph-Like Data Models
Becomes more natural to start modeling your data as a graph when your data become more complex.

graph consists: 
+ vertices (a.k.a. nodes or entites)
+ edges (a.k.a relationships or arcs)

And graph are not limited to homogeneous data,
**an equally powerful use of graphs is to provide a consistent way of storing completely different types of objects in a single datastore.**

ways of structing and querying data in graph.
+ property graph model
+ triple-store model

##### Property Graphs
Can think of a graph store as consisting of two relational tables, one for vertices and one for edges,
Each vertex consists of:
+ A unique identifier
+ A set of outgoing edges
+ A set of incoming edges
+ A collection of properties (key-value pair)

Each edge consists of:
+ A unique identifier
+ The vertex at which the edge starts (the tail vertex)
+ The vertex at which the edge ends (the head vertex)
+ A label to describe the kind of relationship between the two vertices
+ A collection of properties (key-value pair)

Aspect of this model:
+ Any vertex can have an edge connecting it with any other vertex. There is no schema that restricts which kinds of things can or can't be associated.
+ Given any vertex, you can efficiently find both its incoming and its outgoing edges, and thus *traverse* the graph
+ By using different labels for different kinds of relationships, you can store several different kinds of information in a single graph, while maintaining a clean data model.

Provdies great flexibility for data modeling.
Graphs are good for evolvability, a graph can easily be extended to accommodate changes in application's data structures.

##### The Cypher Query Language
*Cypher* is a declarative query language for property graphs.
Query Example:
```
CREATE
  (NAmerica:Location {name:'North America', type:'continent'}),
  (USA:Location      {name:'United States', type:'country'  }),
  (Idaho:Location    {name:'Idaho',         type:'state'    }),
  (Lucy:Person       {name:'Lucy' }),
  (Idaho) -[:WITHIN]->  (USA)  -[:WITHIN]-> (NAmerica),
  (Lucy)  -[:BORN_IN]-> (Idaho)

MATCH
  (person) -[:BORN_IN]->  () -[:WITHIN*0..]-> (us:Location {name:'United States'}),
(person) -[:LIVES_IN]-> () -[:WITHIN*0..]-> (eu:Location {name:'Europe'}) RETURN person.name
```
The Match Query can be read as:
1. **person** has an outgoing **BORN_IN** edge to some vertex. Reach a vertex of type **Location** with **name** property to "United State" by follow a chain of **WITHIN** edge
2. **person** vertex has an outgoing **LIVES_IN** edge. Reach a vertex of type **Location** with **name** property to "Europe" by follow a chain of **WITHIN** edge

We can also start with the two **Location** vertices and work backward.
Thus:
+ Don't need to specify execution details
+ query optimizer will chose the most efficient strategy

##### Triple-Stores
Similar to property model but using different logic language to describe data relationship.
Simple three-part statements:
+ (*subject, predicate, object*)

Subject: equivalent to a vertex in a grpah. The object is one of two things:
1. A value in a primitive datatype. The predicate and object of the triple are equivalent to the key and value of a property on the subject vertex. 
    + For example, (lucy, age, 33) is like a vertex lucy with prop‐ erties {"age":33}.
2. Another vertex in the graph. In that case, the predicate is an edge in the graph, the subject is the tail vertex, and the object is the head vertex. The predicate is the label of the edge that connects them.
    + For example, in (lucy, marriedTo, alain)

##### The RDF Model
The *Resource Description Framework* - a mechnism for different website to publish data in a consistent format. (machine-readable data)

##### The Fouundation: Datalog
Datalog's data model: write it as *predicate(subject, object)*
About *rules*:
+ Define *rules* that tell the database about new predicates (define *predicate*)
+ *predicate* are dervied from data or from other *rules* (like function calling function)

Thus, this provides possibility for building up complex queries.

### Summary

1. New nonrelational "NoSQL" datastores diverged into two main directions:
    + **Document databases** - target use cases where data comes in self-contained documents and replationships between one document and another are rare.
    + **Graph databases** - targeting use cases where anything is potentially related to everything.

2. Each data model comes with its own query language of framework.