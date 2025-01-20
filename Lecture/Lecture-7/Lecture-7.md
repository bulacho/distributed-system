# Lecture 7: Raft (1)

## Topic: state-machine replication
- A popular approach for building fault-tolerant applications
  - Client send operations to primary, primary sequences and sends to backups
  - All replicas execute all operations
  - If same start state, same operations, same order, deterministic, then same end state
  - Example: primary backup in GFS, ops: write/append

- What if primary fails?
  - Coordinator picks new primary in GFS
  - What if coordinator fails?

- Can we have the replicas elect a new primary
  - How about two servers, S1 and S2
  - If both are up, S1 is in charge, forwards decisions to S2
  - if S2 sees S1 is down, S2 takes over as coordinator
  - what could go wrong ?
- The problem: Computer cannot distinguish "server crashed" from "network broken"
  - The symptom is the same: no response to a query over the network
  - this difficulty seemed insurmountable for a long time
  - seemed to require outside agent (a human) to decide when to switch servers
  - we'd prefer an automated scheme!

## Topic: majority rule
- The big insight for coping w/ partition: majority vote
  - have an odd number of servers, e.g. 3
  - agreement from a majority is required to do anything -- 2 out of 3
  - if no majority, wait
  - Why does majority help avoid split brain?
    - at most one partition can have a majority
    - breaks the symmetry we saw with just two servers
  - note: majority is out of all servers, not just out of live ones
  - note: proceed after acquiring majority, don't wait for more since they may be dead
  - more generally 2f+1 can tolerate f failed servers
    - since the remaining f+1 is a majority of 2f+1
    - if more than f fail (or can't be contacted), no progress
  - often called "quorum" systems

- A key property of majorities is that any two intersect
  - servers in the intersection can convey information about previous decisions
  - e.g. another Raft leader has allready been elected for this term

- Two partition-tolerant replication schemes were invented around 1990,
  - Paxios and View-Stamped Replication
  - called "consensus" or "agreement" protocols
  - in the last 15 years this technology has seen a lot of real-word use
  - the Raft paper is a good introduction to modern techniques

## Topic: State-machine replication with raft

- state machine replication with Raft -- Lab 2 + 4 as example:
  - [diagram: clients, 3 replicas, k/v layer + state, raft layer + logs]
  - Raft is a library included in each replica

- time diagram of one client command
  - [C, L, F1, F2]
  - client sends Put/Get "command" to k/v layer in leader
  - k/v layer calls Start() to invoke Raft
    - leader's Raft layer adds command to log
    - leader sends AppendEntries RPCs to followers
    - followers add command to log
  - leader waits for replies from a bare majority (including itself)
  - entry is "committed" if a majority put it in their logs
    - committed means won't be forgotten even if failures
    - majority -> will be seen by the next leader's vote requests
    - leader "piggybacks" commit info in next AppendEntries
  - leader and follower hand commands to k/v layer once entry is committed
    - ApplyMsg and applyCh in lab
  - leader sends response to client

- why the logs?
  - the service keeps the sate machine state, e.g. key/value Db
    - the log is an alternate representation of the same information!
    - why both?
  - the log orders the commands
    - to help replicas agree on a single execution order
    - to help the leader ensure followers have identical logs
  - the log stores tentative commands until committed
  - the log stores commands in case leader must re-send to followers
  - the log stores commands persistently for replay after reboot

- are the servers' logs exact replicas of each other?
  - no: some replicas may lag
  - no: we'll see that they can temporarily have different entries
  - the good news:
    - they'll eventually converge to be identical
    - the commit mechanism ensures servers only execute stable entries

- Implementation challenges:
  - Failures
    - network partitions, lost messages, server crashes
  -Concurrency
    - within a server and between servers
  - Result: many possible executions and many corner cases
    - many details to work -- figure 2
  - Today: electing a new leader, which must handle these challenges

## Topic: leader election (Lab 2A)
- Why a leader?
  - ensures all replicas execute the same commands, in the same order

- Raft numbers the sequence of leaders
  - new leader -> new term
  - a term has at most one leader; might have no leader
  - the numbering helps servers follow latest leader, not superseded leader

- when does a Raft peer start a leader election ?
  - when it doesn't hear from current leader for an "election timeout"
  - increments local currentTerm, tries to collect votes
  - note: this can lead to un-needed elections; that's slow but safe
  - note: old leader may still be alive and think it is the leader

- how to ensure at most one leader in a term?
  - leader must get "yes" votes from a majority of servers
  - each server can cast only one vote per term
    - if candidate, votes for itselft
    - if not a candidate, votes for first that asks
  - at most one server can get majority of votes for a given term
    - -> at most one leader even if network partition
    - -> election can succeed even if some servers have failed
  - note: again, majority is out of all servers

- how does a server learn about a newly elected leader ?
  - the leader sends out AppendEntries heart-beats with the new higher term number
  - only the leader sends AppendEntries 
    - only one leader per term
    - so if you see AppendEntries with term T, you know who the leader for T is
  - the heart-beats suppress any new election
    - leader must send heart-beats more often than the election timeout

- an election may not succeed for two reasons:
  - less than a majority of servers are reachable
  - simultaneous candidates split the vote, none gets majority

- what happens if an election doesn't succeed?
  - no heartbeats -> another timeout -> a new election for a new term
  - higher term takes precedence, candidates for older terms quit

- without special care, elections will often fail due to split vote
  - all election timers likely to go off at around the same time
  - every candidate votes for itself
  - so no-one will vote for anyone else!
  - so everyone will get exactly one vote, no-one will have a majority

- how does Raft avoid split votes?
  - each server adds some randomness to its election timeout period [diagram of times at which servers' timeouts expire]
  - randomness breaks symmetry among the servers one will choose lowest random delay
  - hopefully enough time to elect before next timeout expires
  - others will see new leader's AppendEntries heartbeats and not become candidates
  - randomized delays are a common pattern in network protocols

- how to choose the election timeout?
  - at least a few heartbeat intervals (in case network drops a heartbeat)
    to avoid needless elections, which waste time
  - random part long enough to let one candidate succeed before next starts
  - short enough to react quickly to failure, avoid long pauses
  - short enough to allow a few re-tries before tester gets upset tester requires election to complete in 5 seconds or less

- what if old leader isn't aware a new leader is elected?
  - perhaps old leader didn't see election messages
  - perhaps old leader is in a minority network partition
  - new leader means a majority of servers have incremented currentTerm
  - either old leader will see new term in a AppendEntries reply and step down
  - or old leader won't be able to get a majority of replies so old leader won't commit or execute any new log entries
  - thus no split brain
  -but a minority may accept old server's AppendEntries so logs may diverge at end of old term