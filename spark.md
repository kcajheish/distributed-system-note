# Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing

Spark
- process large data set
- reused working sets across parallel operations
- 10x faster than MapReduce in ML iterative job
- subsecond response time in querying 39 GB dataset

RDD
- resilient distributed dataset
- objects are partitioned across multiple machine
- objects can be rebuilt if machine fails.
- expressibility vs scalability/reliability

lineage
- a way to derive lost partition
- achieve fault tolerance

Spark is implemented and can be extended using Scala.

Spark abstraction
- resilient distributed dataset
- parallel operations

RDD handle
- has info to compute RDD from data in reliable storage

how to construct RDD
- load from file
- transform RDD
    - e.g. flatmap, map, filter
- parallelizing
    - split a scala collection
- change persistence of RDD
    - cache(lazy) vs materizlied in memory on demand
        - cache is not used if out of memory or node fails
    - save

operations: turn files into RDD or RDD into other RDD
- reduce
    - combine elements
    - collected at one process(a driver)
- foreach
- collect
    - sends elements to a driver

shared variables
- updated with closure(map, filter, reduce)
- boadcasted variable
    - send lookup table to the worker once
- accumulator
    - added with associative function and read by a driver

```
# code
lines = spark.textFile("hdfs://...")
errors = lines.filter(_.startsWith("ERROR"))
errors.persist()
```
- after persist, data are cached in memory and can be reused in multiple queries

```
# lineage graph
lines -> errors -> HDFS errors -> timefields

# code
errors.filter(_.contains("HDFS"))
.map(_.split(’\t’)(3))
.collect()
```
- if a partition of errors is lost, rebuild it from lines
- scheduler pipeline filter/map to the node holding cached partitions

DSM(distributed shared memory)
- app can write to a global address space

RDD advantage vs DSM
- use lineage graph to rebuild without checkpoint & program rollback
- mitigate straggler by running the copy of task
- schedule task based on data locality
- scan operation is efficient when memory is not enough

RDD strength
- (o) batch process & analytics
- (x) asynchronous/fine grained update,

```
# ml example: logistic regression
val points = spark.textFile(...)
    .map(parsePoint).persist()
var w = // random initial vector
for (i <- 1 to ITERATIONS) {
    val gradient = points.map{ p =>
        p.x * (1/(1+exp(-p.y*(w dot p.x)))-1)*p.y
    }.reduce((a,b) => a+b)
    w -= gradient
}
```
- sums function of weights and move weight
    - gradient descent

```
# page rank example
val links = spark.textFile(...).map(...).persist()
var ranks = // RDD of (URL, rank) pairs
for (i <- 1 to ITERATIONS) {
    // Build an RDD of (targetURL, float) pairs
    // with the contributions sent by each page
    val contribs = links.join(ranks).flatMap {
        (url, (links, rank)) =>
            links.map(dest => (dest, rank/links.size))
    }
    // Sum contributions by URL and get new ranks
    ranks = contribs.reduceByKey((x,y) => x+y)
    .mapValues(sum => a/N + (1-a)*sum)
}
```


RDD
- each has 1. partition scheme and placement 2. dependency on parent 3. operation
- parent/child dependency
    - wide
        - a parition of parent is sent to many child partitions
        - e.g. join
        - children wait until parents are done and shuffles data
        - slow recovery
            - a failed node leads to recompute of all child nodes
    - narrow
        - a partition of parent is sent to a child partition
        - e.g. map
        - pipeline operation on a node of a cluster
        - fast recovery

HDFS
- iterator: read the block
- preferredLocations: return node that block of a file is on
- partitions: return a partition for each block of the file
