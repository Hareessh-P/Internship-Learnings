# Apache Spark
## Spark vs Hadoop
- Apache Spark does lesser disk IO than Hadoop because Spark stores almost all of its intermediate results in-memory, whereas Hadoop stores them on disk.
- Programmers need not worry about query optimizations.
- If needed, we can code at the lower level; Spark provides access to its RDDs directly and indirectly through DataFrames.
- In MapReduce, data is shuffled on disk, but in Spark, data is shuffled in-memory.
- Spark offers real-time, low-latency processing

## Spark SQL
ref - [Spark Summit (Spark SQL)](https://www.youtube.com/watch?v=AoVmgzontXo&list=RDQM977Pv-p1WSw&index=19)
- DataFrame -> Use DataFrame API (gets converted to SQL queries) -> Query plan generated from SQL queries.
- Series of transformations (Catalyst) done on Query Plan to generate optimized Query Plan.
- Optimized Query Plan converted into low-level code (MapReduce) and executed in the Tungsten execution engine

## Spark
ref - [OSCon on Spark](https://www.youtube.com/watch?v=x8xXXqvhZq8&list=RDQM977Pv-p1WSw&index=30)
- Distributed programming operations: Broadcast, Take (Expensive, we have to be careful as the Driver is the bottleneck), DAG (transmitted from Driver to executor, inexpensive), Shuffle (required in join, group by, reduce).
- ![alt text](https://github.com/Hareessh-P/Internship-Learnings/blob/master/pics/Pasted%20image%2020240605102837.png)
- The number of shuffle objs is almost the product of the number of map and reduce tasks.
- 100 map and 100 reducers, then we'll have 10,000 intermediate files. All these files have to travel through the network, they all have to get serialized and deserialized, and they might need disk IO too because mostly they can't fit into memory.
- Long story short, Shuffle is the most expensive operation.
- Exchange of data or state is expensive and should be done in a lockstep manner.
- Estimation reference - [Spark Performance Optimization Guidelines](https://developer.ibm.com/blogs/spark-performance-optimization-guidelines/#:~:text=Formula%20recommendation%20for,cores%20*%20total%20cores).

### RDD (Resilient Distributed Dataset)
- Set of immutable data, should be fault-tolerant to exceptions, achieved by the immutability of the data and replay
- No schema in RDDs, but with the DataFrame API introduced, we can enforce schema 

### DAG - actions and transformations
- Flow of the execution, if any step has failed, then it would refer to the DAG (placed in-memory) and recompute that.
- Before Spark, DAGs weren't a part of execution engine. Quick lookup is a win here.

#### Before Spark :

![alt text](https://github.com/Hareessh-P/Internship-Learnings/blob/master/pics/Pasted%20image%2020240605114329.png)
	- input -> 1 mapper, 1 shuffle, 1 reducer -> output :--> so potentially only one join can be done..
	- so we have to chain many of these
	- shuffle and output required disk io
	- map, reduce required spinning up and down the clusters (aka YARN containers)

#### After Spark :

![alt text](https://github.com/Hareessh-P/Internship-Learnings/blob/master/pics/Pasted%20image%2020240605114403.png)
	- we have to just spin up YARN containers that would chain the map and reduce tasks and if we organize mem correctly we needn't hit the disk throughout the process
	
![alt text](https://github.com/Hareessh-P/Internship-Learnings/blob/master/pics/Pasted%20image%2020240605114904.png)
	- Now why don't we do this for all the query at once? This introduced *Spark Streaming*
	- Spark SQL is used as a service where we leave the containers up (that is we keep them live) and with proper mem mngt we can exe all the queries without spinning up and down

### Dynamic Allocation --> TODO: dig in 

### Simple Code : 

![alt text](https://github.com/Hareessh-P/Internship-Learnings/blob/master/pics/Pasted%20image%2020240605113234.png)
- C - setting up context , T - transformation, A - Actions
- explicit objects as funcs are sent through the wire(network) and getting invoked in the worker nodes
- these objs get serialized and un-serialized 

### Managing Parallelism (Some issues):
- Fixed and Dynamic Allocation
- ### Skew :
	- even though we have written a distributed process, all our execution is almost executed on a single core..
	- 1 case to be careful : Join of 2 table where *NULL* entries(80%) r there, so here when we join all the *NULL* keys and shuffle(partition based on the key) all of them go to the same node (80% of data). :/
	- Easy check for skewness : if one of our nodes takes more time than the others then we should look into the issue related to skew
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
