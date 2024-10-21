# ClickHouse - Lightning Fast Analytics for Everyone

ClickHouse
- Handle concurrent requests against petrabyte of data.
- High ingestion rate achieves real time latency.

Table is sharded based on shard expression
- Each shard is stored in different node.
- Enable load balancing & parallel processing.

Replication is acheved with Keeper.
- Using Raft consensus algorithm.

A table has many part.
- Each part is created when table is inserted.
- Rows in a part are sorted by primary key.
- During compaction, many parts are merged using k-way merge sort; source part is removed as soon as reference count is zero(no query against the part).

async vs syn insert
- In async insertion, rows are saved in buffer in which data are used to create part.
    - Higher throughput at the risk of loss of data.
- In sync insertion, rows are saved in part directly.
    - Higher data integrity but merge overhead is large.

Part
- Directory + files for each column.
- In each part, rows are sorted by primary key.
- Include many granules.

Granules
- smallest unit for scan and index lookup
- Number of records in granules is limited to 8192 records. This increase locality for read and write.

Data are written in the unit of block(1MB) which has many granules.
- To save space, data is compressed before written to disk and decompress before loading into memory.

Sparse index
- Use 1000 index to binary search 8.1 million rows.
    - primary key of first row in granule: granule id
    - granule id : block id

Project
- Additional table that sort rows on a different columns
- Enable efficient lookup at the cost of write overhead, space, and merge.

skipping index block
- Scan block statistics to skip even more rows than granule.
- e.g. min, max

merge type
- replacing merge
    - keep the most recent(latest inserted timestamp) rows
- aggregating merge
    - combine many rows with same pk into one
- TTL merge
    - can move historical data into cold storage(e.g. S3)

three ways to change the data
- Mutate the data in place
    - con: select is not atomic
- Lighweight delete use a bitmap to remember which rows are deleted and apply bitmap
as a addition filter to query.
    - con: increase select cost
- Serializing
    - con: only applicable when update is rare

idempotent
- Client issues an insert but disconnects from network before getting a response. We need a mechanism to avoid duplicated insertion.
- When a row is inserted, part is hashed and saved in Keep.
- In the Keep, only last N hashes are kept.
- To avoid duplicate insert, lookup hashes in Keep.

Replication log records operation on a single node
- Raft is used to spread logs to other nodes.
- Replicated table is eventually consistent.

Query are executed against a snapshot.
- A read won't be affected by concurrent writes.
- A reference count tracks num of read to a part. Parts are removed after read finishes

Write are batched and are not forced to commit.
- Increase write performance and sacrifice atomicity.

Parallel processing
- Table is shared across many nodes.
- Each node can execute operators on many threads.
- Each operator can executes many rows at a time.

Queries are classified into many workload class.
- Each class has limit on resource(cpu, memory, IO...etc).
- All types can make progress without competing with each other.

Limit number of threads working on a query.
- It avoid oversubscriptions when there are many concurrent queries.

Integration: collects data from external source.
- (x)push: large architectural footprint; scalability bottleneck
- (o)pull: join between local & remove, simple architecture, reduce time to the insight
