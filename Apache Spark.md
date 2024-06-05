# Apache Spark
## Spark vs Hadoop
- Apache Spark does lesser disk IO than hadoop , cuz spark stores almost all of its intermediate results in-memory.. but the hadoop stores it in the disk
- programmers need not worry abt query optimizations.
- If needed we can code at the lower level , that is spark provides access to its RDDs directly and indirectly through (Dataframe)
- in map reduce data is shuffled in disk but in spark data is shuffled in-memory.
- real-time , low-latency 
## Spark SQL
ref - [Spark Summit (Spark SQL)](https://www.youtube.com/watch?v=AoVmgzontXo&list=RDQM977Pv-p1WSw&index=19)
- dataframe --> use df api (gets converted to sql queries) --> Query plan generated from sql queries
- Series of transformations (Catalyst) done on QP --> to generate optimized QP 
- OQP converted into low level code (map-red) n gets executed in the Tungsten execution engine

## Spark
ref - [OSCon on Spark](https://www.youtube.com/watch?v=x8xXXqvhZq8&list=RDQM977Pv-p1WSw&index=30)
- Distributed programming operations : Broadcast, Take(Expensive, v have to be careful-->Driver is the bottleneck), DAG (transmitted from Driver to executer, inexpensive), Shuffle (req in join , grp by, reduce)
- ![[Pasted image 20240605102837.png]]
- no of shuffle objs is almost the product of no of map n reduce tasks
- 100 map n 100 reducers , then we'll have 10,000 intermediate files --> all these files have to travel through the network, they all have to get serialized n de-serialized.. n might be they may need disk io too cuz mostly they cant fit into the mem..
- long story short Shuffle is the *most expensive* operation..
- exchange of data or state is expensive n shd be done in a lock step manner
- estimation ref - https://developer.ibm.com/blogs/spark-performance-optimization-guidelines/#:~:text=Formula%20recommendation%20for,cores%20*%20total%20cores.
### RDD (Resilient Distributed Dataset)
- set of immutable data, shd be fault tolerant to exen, thats achieved by immutability of the data n replay
- no schema in rdds but with the dataframes api introduced we can enforce schema 
### DAG - actions n transformations
- flow of the exen, if any step has failed then it would refer the dag(placed in-mem) n recompute that.
- Before spark, DAGs weren't a part of execution engine (ig the quick lookup is a win here..)
#### Before Spark :

![[Pasted image 20240605114329.png]]
	- input -> 1 mapper, 1 shuffle, 1 reducer -> output :--> so potentially only one join can be done..
	- so we have to chain many of these
	- shuffle n output required disk io
	- map, reduce required spinning up n down the clusters (aka yarn containers)
#### After Spark :

![[Pasted image 20240605114403.png]]
	- we have to just spin up yarn containers that would chain the map n reduce tasks and if we organize mem correctly we neednt hit the disk throught the process
	- 
![[Pasted image 20240605114904.png]]
	- Now why dont we do this for all the query at once ?.. so this introduced *Spark Streaming*
	- Spark sql is used as a service --> where we leave the containers up (that is we keep them live) n with proper mem mngt we can exe all the queries without spinning up n down.. 

### Dynamic Allocation --> TODO: dig in 

### Simple Code : 

![[Pasted image 20240605113234.png]]
- C - setting up context , T - transformation, A - Actions
- explicit objects as funcs are sent through the wire(network) n getting invoked in the worker nodes
- these objs get serialized n un-serialized .. 

### Managing Parallelism (Some issues):
- Fixed n Dynamic Allocation
- ### Skew :
	- even though we have written a distributed process, all our execution is almost executed in a single core..
	- 1 case to be careful : Join of 2 table where *NULL* entries(80%) r there, so here when we join all the *NULL* keys n shuffle(partition based on the key) all of them go to the same node (80% of data). :/
	- Easy check for skewness : if one of our nodes take more time than the others then we shd look into the issue related to skew
	- #### Adding Salt to our Keys :
		- that is say we r processing stocks data, apple stocks r more in no , then we can add salt/dirt to the keys we pick randomly the apple stocks n mark them as apple-1 n apple-2 --> so that we use 2 nodes to process apple stocks..
	- #### Handling Cartesian Joins : 
		- Nested Structure
		- Windowing
		- ReduceByKey
	- Repartitioning can be done if some stage is more cpu intensive...
	- repartitioning involves shuffle  --> expensive

## Performance Optimizations in Spark :

### Lazy Loading : 
- transformations are tracked (not executed as flow), when an *action* is called out all the transformations are made (by optimizing the trans..) 
- So *actions* must be called only when needed.
### File Formats :
- Usually column based stores provide boost to the performance.(ref - DB internal pg 34)
- Spark has *Vectorization support* --> col stores --> Reduce Disk io significantly.
- #### Small files issue : 
	- when dealing with lot of small files --> open , read, close files need disk io
	- spatial locality is not being achieved => less performance
	- solution ==> File Compaction
	