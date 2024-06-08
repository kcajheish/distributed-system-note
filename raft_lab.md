# Raft
Raft is a consensus algorithm that elects leader and replicates logs. The key challenge is to have every server owns the same copy of logs. These logs are then applied to state machine of the server. Each server has the same states and can be picked as the leader when current leader fails. Its design achieves fault tolerance, availability, and safety.

In raft, server is in candidate, leader or follower state.

Leader
- Candidate server becomes leader by winning the most votes from follower in current term.
- Leader maintains authority by calling AppendEntries RPC.

Candidate
- Follower transitions to candidate when it doesn't receive AppendEntries from leader for a duration, election timeout.
- Candidate calls RequestVote RPC to all peers.

Follower
- Follower is initial state of all servers.
- Follower receives RPC calls from leader or candidate. It never makes call to candidate and leader.

Safety
- once logs are commited, system won't lose it. When leader fails, new leader can be elected without losing data or return wrong data to the client. To achieve safety, raft does the following
    1. check consistency while leader replicates logs to follower
    2. while candidate requests logs, check whether logs are up to date

Log compaction
- once logs are applied to state machine, it can be removed from durable storage so that space is enough for new logs.

Automated configuration change
- Adding/removing server should be part of the algorithm. This reduces manual errors.
- After configuration is given to leader, leader creates an log entry sepcify the config changes. Once that log entry is commited, leader of the new configuration will take the lead.

Snapshot
- Leader copies state machine to snapshot. Leader calls InstallSnapshot RPC and send snapshots to lagging follower.
- Each server maintains its own snapshot. This reduce network bandwidth where leader needs to send those snapshot to follower.
