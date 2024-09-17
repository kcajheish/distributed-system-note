# Kafka: a Distributed Messaging System for Log Processing

## persistence

data is written to persistent log rather than is cached and flushed

- kernel page cache is efficient
  - os does better job maintaining coherence between pagecache and disk
  - compact bytes without overhead
  - no gc penalty
  - warm cache after a crash
- sequential read/write is fast
  - optimization
    - read ahead
    - write behind
  - for sequential, 600 MB/s
  - sequential > random memory access in some cases

per-consumer queue

- maintain metadata message
- btree
  - 10 ms for a seek
  - seek can't be paralleled
  - random access
- persistent queue
  - O(1) operation
    - read
    - append
  - retend message after consumption

## efficiency

byte copy

- when you read data from disk and send data to the network, it goes through
  - disk -> kernel pagecache
  - kernel pagecache -> user space buffer
  - user space buffer -> socket buffer
  - socket buffer -> NIC buffer
- that's a lot of copies and not efficient; thus, use sendfile
  - it copy data from kernel pagecache to NIC buffer directly

many small IO

- batch request for each message set
- linear chunks of msg are appended to log and fetched by consumer

end to end batch compression

- to reduce network bandwith usage, compress a batch of message
- messages are decompressed when consumer processes the messages

## producer

load balance

- producer publishes messages to the leader of partition
- partition is determined by hash of the key. e.g. user_id
- if producer connects to the wrong node, node returns address of the specific partition node so that producer can be redirected

async read

- requests are batched in memory
- requests are sent after a period or buffer being full
- it trades latency with throughput

# consumer

consumer fetch messages from broker by

- specifying offset/partition
- note, broker is the leader of the node

choose pull over push

- lagging consumer can catch up when it can
- broker doesn't have to backoff pushing based on consumer's status
  - it is not easy

use long pulling

- cause: broker has no new data; consumer pulls in a tight loop
- consumer is blocked until new data is arrived

how about using pull for producer?

- e.g. producer writes to local disk; broker pull from producer local disk
- it's not reliable to maintain persistent storage over thousands of producers

topic -> multiple partition -> each consumed by a consumer in a group

- broker maintains state of current offset
- offset is checkpointed for reliability
