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

# Lab A

Cases of leader election
1. All servers start as followers. Eelect a leader.
2. Leader maintains authority.
3. Leader fails, and new leader is elected.

When a candidate send a RequestVote RPC to a server:
1. If current server term is higher than candidate, don't give vote. Return the term so candidate can transit to follower and update its term.
2. If current server is an outdated leader, its term is lower than the candidate. Leader transits to follower and grants vote.
3. If current server hasn't voted or already voted for the candidate, grant vote.
4. If current server voted for other candidate, don't grant vote.
5. Follower has to remember candidateId that it votes. When another candidate requests for votes, it can reject the vote if id doesn't match.

When leader sends AppendEntries RPC to a server
1. If term of the server is higher, reject the request so that leader can update term and transit to follower.
2. Health check succeeds.
    - Update timeout so server won't keep becoming candidate.
    - Update the server id that server votes for. Also, update the term and transit into follower status. This ensure obselete leader follows new leader.
    - Randomize timeout so followers won't turn into candidate at the same time.

**ticker** is called by follower periodically.
1. Follower becomes candidate after timeout.
    - Timeout is updated when leader calls AppendEntries.
2. Candidate calls RequestVote to all peers in parallel.
    - If RequestVote fails, retry.
    - Collects votes. Lead when candidate wins majority.
    - Exit early if
        1. candidate becomes leader
        2. follower returns with higher term
            - Candidate returns to follower and update the term.
        3. timeout
3. Holds the lock. We don't want candidate changes to other status when it is collecting votes.

**heartbeats** is called by leader.
1. Leader sends AppendEntries to all peers for every 0.1s.
    - if AppendEntries fails, retry
    - if AppendEntries returns higher term, leader transits to follower.
2. Holds the lock so that leader won't change to other status when sending heartbeats.
