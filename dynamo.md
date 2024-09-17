## Dynamo: Amazon’s highly available key-value store

- trade consistency with availability
- use object versioning and conflict resolution
- key value storage

use gossip to detech failure, membership protocol

to manage consistency

- decentralized replica synchronization
- quorum like technique

to replicate

- consistent hashing

results

- 3M checkouts/day
- 10M cart/day

design consideration

- eventual consistent data store
- conflict resolution
  - last write win, executed by datastore
  - conflict merge, executed by app who knows schema
  - always writable
    - read has to resolve conflicts as well
  - write to majority
    - write is rejected if it can't reach majority of nodes
    - achieve CP
- incremental scalability
  - minimized impact during scale out
- symmetry
  - each node has same responsibility
  - simplify provision and maintenance
- decentralization
  - e.g. peer to peer
  - avoid outage
- heterogeneity
  - load balance based on host capacity
  - don't have to upgrade hosts when a new node is provisioned

core techinique

- partitioning
- versioning
- membership
- failure handling
- scaling

get(key)

- locate replica associated with key
- return a list of objects, each having a version

put(key, context, object)

- context is used to validate meta data of the object; it includes version
- key/object are treated as array of bytes in dynamo

to determine node for a key

- apply MD5 to the key
- check position on the ring

challenge of consistent hashing

- nodes are not assigned on the ring evenly -> uneven load
- didn't consider heterogeneity

to address these, use virtual node

- create tokens on the ring; each token = virtual node
- each node is resopnsible for more than one virtual node
  - actual number depends on its capacity -> handle heterogeneity
- load is evenly distributed when a node is added/removed

coordinator node replicates key to other node

- clockwise, N-1 nodes which are picked from preference list

data versioning

- update to an older version should be preserved when newest version is not available now
- version is diverged when network partitioned and node fails
- vector clock
  - a list of [node, counter]
- both version of data are kept if no casual relation between both
- vector clock size can grow rapidly during large server outage
  - truncate the vector when size reaches a threshold(e.g. len(preference list))

Quorum like system

- R+W > N
  - R nodes participate in read
  - W nodes participate in write
  - N num of replicas

hinted handoff

- A node fails and can't process write request. Another node take over the requests and store the request in local storage. The request is sent back when failed node recovers

In anti-entropy, use merkle tree to detect inconsistency due to permanent failures.

- leaf, hash of key in a range(determined by the virtual node)
- parent, hash of two children
- two replica compare hash value of the node; if they are not the same, some key is out of synced.
- pro
  - Check consistency node by node rather than entire tree; hence reduce network usage in network bandwidth
- con
  - have to recalculate merkle tree when a node is removed/added the the cluster

Decentralized failure detection

- nodes are notified when a node is removed/joined
- can't communicate with failed node

add/remove storage node

- even distribution of key -> fast bootstrap, low latency
- confirmation round -> no duplicate transfer

read repair

- node that returned stale data will be fixed during read.

read your write

- coordinating node is picked from top n, whose is first to reply in previous read requezt

membership

- to add/remove node, issue change of membership request
- A node receives request and persist it in its local
- A node pick a peer randomly, and resolve their membership difference by gossip protocol
  - i.e. know its peer token
