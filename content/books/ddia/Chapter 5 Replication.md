+++
author = "Ping Zhang"
title =  "Chapter 5 Replication"
date = "2024-01-14"
description = "design data intensive application"
tags = [
    "ddia",
    "system"
]
series = ["ddia"]
+++

# Replication

- Shared memory architecture -> all the components can be treated as a single machine
- Shared disk architecture --> uses several machines with independent CPUs and RAM

Assume that your dataset is small enough to fit into a single machine. Mainly three ways to replicate changes between nodes:

1. single leader (getting all the nodes to agree on a new leader is a consensus problem)
2. multi leader
3. leaderless
Tradeoffs:
- sync or async replication
    - `sync`: it is impractical to sync with all followers, as if one follower fails, it would block the whole system
    - Often, leader-based replication is configed as `async`.
- how to handle failed replicas

## 1. Single Leader Replication

### Setup a new follower

take a consistent snapshot without --> follower gets it --> sync with leader about changes since last snapshot (mysql --> binlog)

### Potential issues with leader auto failover

1. if using async replication, the new leader may not have received all the writes from the old leader. When the old leader rejoins the system, it might have different data.
2. Discarding writes is dangerous if other storage systems outside the database need to be coordinated with the database content. E,g. pkey of a table is stored in redis as cache.
3. Split brain issue. there could be two leaders at the same time
4. Need a good timeout value to determine if the leader fails. A longer timeout leads to longer time to recover. A short timeout can lead to unnecessary failovers.

### Implementation of replication log

1. `Statement based replication`. Leader logs every request (statement) that it runs and sends it to followers. However,
    1. for some nondeterministic func, like `NOW(), RAND()`, which could cause issues
    2. Statements with side effects (e.g. `triggers, stored procedures`) may result in different side effects unless they are deterministic
    3. Some statements must be executed in the same order (e.g auto-incrementing cols). This could be limiting when there are many concurrently running transactions.
2. `Write ahead log (WAL) shipping`: `log` is an append-only sequence of bytes containing all writes to the db. (Psql and Oracle use this way)
    1. cons: log describes data on a very low level: a `WAL` contains details of which bytes were changed in which disk blocks, which makes it coupled to storage engine. If the storage format is changed between engine version, this wont work.
3. `Logic (row-based) log replication`: it decouples from storage internals. It is useful if you want to send the contents of a data base to an external system, e.g. data warehouse for offline analysis. this is called **change data capture**
4. `Trigger-based replication`: (like triggers and stored procedures in many relational databases)

### Problems with replication lag

`Read scaling arch`: have one leader takes write, many clients connect to followers when the app is read heavy. This can only work with **async** replication. Downside: with **replication lag**, clients might get inconsistent reads, **eventual consistency**, but there is no limit to how far a replica can fall behind, especially with network issues or system is under heavy load.

Some problems and solutions:

1. Reading your own writes
This is called `read-after-write consistency` or `read-your-writes consistency` . It becomes complex to provide `cross-device read-after-write consistency`.
2. Monotonic reads
**Problem**: Anomaly can occur when reading from async followers, users could see things moving backward in time (inconsistent result, first read from a replica with more recent result, the second read from a replica with stale result).
Solution: `Monotonic reads`: it is a lesser guarantee than strong consistency, but stronger than eventual consistency. It guarantee if one user makes several reads in sequence, they will not see time go backward. One way to achieve is ensure each user always makes their reads from the same replica, e.g. the replica can be chosen based on a hash of the user ID, rather than randomly chosen).
3. Consistent Prefix Reads.
**Problem**: violation of causality: A asks a question and B answers. when there is lag, a user can read B's answer first, leading to confusion. Solution: `Consistent prefix reads`: guarantees that if a sequence of writes happens in a certain order, anyone reading those writes will see them in the same order. This is extremely true for distributed databases, different partitions operate independently, so there is no global ordering of writes. A potential solution is to keep any writes that are causally related to the same partition.

## 2. Multi-Leader Replication

**It rarely makes sense to use a multi-leader setup within a single datacenter, since the benefits rarely outweigh the added complexity**

Each datacenter has a leader-follower setup. Benefits:

1. inter-datacenter network delay is hidden from users
2. tolerance of datacenter outages
3. tolerance of network problems
Downside: need to handle **conflicts**, auto-incrementing keys, triggers, integrity constraints can also be problematic.
Examples similar to multi-leader replication:
4. clients with offline operation: e.g: calendar app, which has a local database on the phone
5. collaborative editing: when a user changes a doc, the changes are instantly applied to their local replica and asyncly replicated to the server and any other users who are editing the same doc.

### How to handle conflict

1. synchronous conflict resolution. a leader writes data and waits for the write to be replicated. However, this defeats the purpose to allow each leader to accept writes independently.
2. Avoid conflict.
3. Converging toward consistent state, e.g.
    1. given each write a unique ID and use `last write wins` , but it can lead to data loss.
4. Custom conflict resolution logic. Delegate the responsibility to the application itself to resolve conflict. The code can be executed on read (multiple versions of data are returned and let application handle) or write (called `conflict handler`).

### Multi-leader replication topology:

This describes the communication paths along which writes are propagated from one node to another.

1. Circular topology
2. Star topology
3. All to all topology.
In circular and star topology, if a node fails, it can interrupt the flow of replication message between other nodes. In all to all, in some cases, some network links could be faster than others, leading to replication messages overtake others. This can cause problem of causality.

## 3. Leaderless Replication

e.g. `Dynomo`, `Riak`, `Cassandra` and `Voldemort` . E.g. implementation:

1. the client directly sends its writes to several replicas
2. a coordinator node does this on behalf of the client.
A client writes to all nodes and considers the write to be successful if it reaches quorum. Reads are also sent to several nodes in parallel, which could return many different versions. Thus version numbers are used to determine which value is newer.

### How does a node catch up on the writes that it missed during its outage

1. Read repair: when a client reads from several nodes in parallel, it can detect which node is stale. The client then fixes the stale value. This works well for values that are frequently read.
2. Anti-entropy process: have a background process that looks for difference in the data between replicas and fix them

### Quorums for reading and writing

`n` replicas, `w` nodes to confirm writes, `r` nodes to query for each read: as long as `w + r > n`, we expect to get an up to date value when reading, since at least one of the `r` nodes must be up to date. --> `quorum reads and writes` (in which nodes you write and read overlap)
Common choice: `n` is odd number and set `r = w = ceil((n + 1) / 2)`
Normally, reads and writes are sent to all n replicas in parallel. The parameter `r` and `w` determine how many nodes we wait for.

### Limitation of Quorum reads and writes

1. if a write happens concurrently with a read, the write may be reflected on only some of the replicas. In this case, it is undetermined whether the read read returns the old or new value
2. If a node carrying a new value fails, and its data is restored from a replica carrying an old value, the number of replicas storing the new value may fall below `w`, breaking the quorum condition.
3. No good monitoring for data staleness, especially for `read repair`. (since for leaderless replication, there is no fixed order in which writes are applied)
Dynomo-style databases are generally optimized for use cases that can be tolerate eventual consistency.

### Sloppy quorums and hinted handoff

Quorums are not as fault-tolerant as they could be. A network interruption can easily cut off a client from a large number of db nodes, thus clients that is cut off can no longer reach a quorum, they are essentially dead.
**Sloppy quorum**: writes and reads still require w and r successful response, but those may include nodes that are no among the designated n home nodes for a value.
**Hinted handoff**: once network interruption is fixed, any writes that one node temporarily accepted on behalf of another node are sent to the appropriate home nodes.

### Detecting concurrent writes

Problem: events may arrive in a different order at different nodes due to network delays and partial failures. If each node simply overwrote the value for a key whenever it received a write request, the node would become permanently inconsistent. Unfortunately, the most implementations cannot handle this, it relies on the application developer need to know a lot about the internals of the db's conflict handle.

1. last write wins (discarding concurrent writes), as long as we have some way of unambiguously determining which write is more "recent". and every write is eventually copied to all replicas. Even thought the writes dont have a natural ordering , we can force an arbitrary order on them. e.g. attaching a timestamp to each write, pick the biggest one. LWW is at the cost of **durability** since some reported successfully writes are discarded by the system.
2. Happens-before relationship and concurrency. **Whether one op happens before another op is the key to defining what concurrency means.** (it is not important whether they literally overlap in time, because of problems with clocks in distributed systmes)
3. Capturing the happens-before relationship.
    - Single replica example:
        1. The server maintains a version number for every key, increments the version every time that key is written, stores the new version number along with the value written.
        2. A client must read a key before writing. the server returns all values that haven't been overwritten along with latest version number.
        3. When writes a key, the client must include the version number from prior read and merge together all values it received.
        4. (with version, the server knows when writes without version numbers are concurrent, since they are first op.)
        - The client should include logic to merge concurrent written values. The server side could have a deletion marker (`tombstone`) so that the client knows it is deleted.
    - with more replicas, each replica needs to main this version number for a key. The collection of version numbers from all replicas is called a `version vector`.