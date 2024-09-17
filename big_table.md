# Bigtable: A Distributed Storage System for Structured Data

What is BigTable?
- a storage/database system that holds terabytes of data.
- low latency, high throughput, availability
- dynamic control of data format
- 1 terabyte = 1024 GB

Structure of table
- row key, column key = column family: qualifier
    - e.g. language:ID, anchor:host
- Rows are sorted by row key.
- Each cell has multiple version of data. They are indexed by timestamp.
- number of columns are unbounded

For example,  A Web table has
- row key: url
- column family: anchor (who reference the content)
- column data: content


Read/write to a row is atomic. It’s easier for developer to make sense of the behavior
- e.g. For a read modify write, the read data isn’t changed after write is committed.

how to avoid old data?
- garbage collect outdated data. For example,
- keep last n version
- keep version in n days

machine
- GFS stores file and logs
- cluster management for machine status, failure, resource
- Table is partitioned to a shared pool of machine

SSTable data structure
- Store key:value pair
- It’s a sequence of blocks(~64KB). Each block has index associated with it.
- To read the block for a key, binary search and find the block for the key

Chubby
- distributed lock service
- use paxos for replication(master-slaves)
- read/write to the directory requires lock and is sequential.
- have directories and small files
- store live tablet servers
- store Bigtable schema

Client
- sessions acquires lock and handle from Chubby. It has to renew lease before session expired
- client read tablet metadata from master server and cache it
- client read tablet data from tablet server
- master load is reduced
- preferetch the tablet location and cache it for lookup

Master
- distribute tablet data to tablet server
- add/remove server from cluster
- balance load on tablet server
- maintain schema e.g. table, column family…etc
- garbage collect file in GFS

Tablet server
- read/write to a tablet(~100 MB)
- holds ~1000 tablets
- split table when table grows too big


Table: a set of tablet

Tabet: rows within a key range

Three level hierarchy
- Root metadata table
    - in files and chubby
- metadata table
    - in memory
location for (table_id, end_row), logs to each tablet, user table
- in tablet server
- holds data for ~1000 tablets

Master for tablet
- assigned tablet to tablet server with free space.
- tracks live servers, current assigned tablet, unassigned tablets.

When a tablet server starts, it creates a unique file for lock in server directory. Tablet server acquires lock in a session. Then it starts serving the traffic. Tablet server has to renew the lease to update its lock status. If it fails to do so, unique file will be removed by the master

Master keeps asking tablet server for lock status. If tablet server loses lock or can’t respond, master acquires lock for the same file in chubby and delete it.

When master for table starts
- it acquired lock in chubby
- read live servers in server directory
- ask live tablet server for the tablet they have
- master scan tablet metadata for any unassigned tablet

Memtable stores recent commit log.
- SSTable is in GFS. It has old commit log.
- Each write to a tablet server is appended to a log file for that server.
- Batch write -> group commit
- A sequence of SSTables + Memtable -> merge view in sorted order for read

minor compaction
- Each commit log is first applied to Memtable. When Memtable size reaches a threshold, some of the logs are moved to SSTable in GFS.

merging compaction
- merge Memtable and several SSTable into a new SSTable

major compaction
- merge Memtable and all SSTables into a SSTable. It removes all deleted entries and reclaims resources used by deleted data.

Split the data since you don’t have to read everything.
- e.g. you don’t have to read web content if you only need metadata.
- Several column families form a locality group. Each locality group for a tablet is stored in a SSTable.
- Lazily load a locality group into memory for frequent read data

Use bloom filter to minimize number of disk seeks.
- Read every SSTable for a merge view. We use bloom filters to know whether a SSTable contains the column/row we are looking for.

Blocks in SSTable for a locality group can be compressed. Use two passes scheme.
- first pass: compress common string
- second pass: compress repetition

Caching for read performance
- first level cache: data returned by SSTable interface
	- fast for frequent read data
-  second level cache: SSTable blocks returned by GFS
    - fast for read data that are closed.
- e.g. fast sequential read

Each tablet server has one commit log. why not one commit log for each tablet?
- many concurrent reads
- ineffective group commit(batch write)

commit logs to gfs can be slow(e.g. high latency)
- Use a backup thread. When committing logs takes too long, switch to another thread and commit logs to a different log file.
