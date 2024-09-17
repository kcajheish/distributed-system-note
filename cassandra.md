# Cassandra - A Decentralized Structured Storage System
hash function is used to partition data
- each partition is replicated to multiple nodes
- distribute load evenly to avoid hot partition
- each replica for a partition can accept update to a key.
- use last write win to resolve conflicts when a key is concurrently updated in multiple replica

consistent hashing
- each node holds tokens which map to different range of keys(token ranges);
- to know which node a key maps to
  1. hash(key)
  2. find its position on the ring
  3. walks clockwise until RF tokens is met
  4. store the key to the node that token maps to
- RF: replication factor
  - num of replicas holds a copy of partition
  - to achieve availability, replica must be in different physical nodes

virtual nodes
- resolve key imbalance in nodes when a node is added to the ring
- term
  - token: a unique position on the ring
  - host id: identifier of physical node
  - endpoint: ip of physical node
  - virtual node: tokens owned by same physical node
  - token map: token, host ip
- pro
  - you can add/remove a node from the ring and still keep data in nodes balanced
  - load is distributed evenly on nodes

replication strategy
- every keyspace has its own strategy
- determine which node is replica for a token range

network topology strategy
- allows picking nodes as replica from different rack in a datacenter

simple strategy
- pick nodes up to replication factor for a token range
- it ignores any configured datacenter and rack

Data Versioning
- Cassandra uses mutation timestamp versioning to guarantee eventual consistency of data
- Concurrent updates resolve according to the conflict resolution, last write wins
- Cassandra’s correctness depend on these clocks. Use NTP to synchronize time
- timestamps is applied to every column of every row

Replica Synchronization
- As replicas in Cassandra can accept mutations independently, it is possible for some replicas to have newer data than others.
- best-effort techniques to drive convergence of replicas
  - Replica read repair in the read path
  - Hinted handoff in the write path.
- anti-entropy repair
  - hash trees over their datasets called Merkle trees
  - compared across replicas to identify mismatched data
  - ensure eventual consistency
    consistency level
- R + W > N
  - ensure strong consistency
  - pick read and write consistency levels such that the replica sets overlap, resulting in all acknowledged writes being visible to subsequent reads
  - If this type of strong consistency isn’t required, lower consistency levels like LOCAL_ONE or ONE may be used to improve throughput, latency, and availability.
    - e.g. smaller R, W
- write:
  - how many responses the coordinator waits for before responding to the client
  - note: write operations are always sent to all replicas
- read
  - the coordinator generally only issues read commands to enough replicas to satisfy the consistency level

Distributed Cluster Membership and Failure Detection
- Gossip
  - liveness information is shared in a distributed fashion through a failure detection mechanism based on a gossip protocol.

## storage engine

every write to a node is appended to a commit log
- commit log is shared by multiple table
- commit log is durable and can be used for recovery
- a new segment of commit log is created once current segment of commit log exceeds a threshold
- commit log is recycled once data are flushed to disk(SSTable)

memtable
- write back cache
- a write to commit log and memtable
- in a node, each table has a memtale for its partition
- data in memtable are in sorted order
- once memtable reaches a size threshold, data in memtable is fluhed to a immutable

SSTables
- immutable data files that persists data on disk
- a SSTables per table
- a partition is stored across multiple SSTable files, as data is added or modified
- As data are flushed to disk from memtables or are streamed from other nodes, Cassandra compacts multiple SSTables into one. Once the new SSTable has been written, the old SSTables can be removed.
- rows in one partition is ordered by clustering keys

files
- Partitions.db: partition index file maps unique prefixes of decorated partition keys to data file locations
- Data.db: the contents of rows
- Rows.db: map each row key into the first block where any content with equal or higher row key can be found

## guarantees

Cassandra prioritize availability and parition tolerance over consistency

high scalability
- can add/remove nodes

durability
- data is copied to many nodes

eventual consistency
- trade availability with latency & consistency
- use last write win in read repair

secondary index
- consistent with local replica
