+++
author = "Ping Zhang"
title =  "Chapter 6 Partition"
date = "2023-12-29"
description = "design data intensive application"
tags = [
    "ddia",
    "system"
]
categories = ["software"]
+++

It is called `shard` in MongoDB, `region` in HBase, `tablet` in Bigtable, `vnode` in Cassandra and Riak, `vBucket` in Couchbase.

# Covers:

- Different approaches to partition
- how the indexing of data interacts wit partitioning
- rebalancing
- how databases route requests to the right partitions and execute queries

# Partitioning and Replication

Even each record belongs to exact one partition but it can still be stored in many nodes for fault tolerance. A node may store more than one partition. In a leader-follower replication model: Each partition's leader is assigned to one node, and its followers are assigned to other nodes. Each node may be the leader for some partitions and a follower for other partitions.

# Partitioning of Key-Value Data

The presence of skew makes partitioning much less effective.

## 1. Partitioning by key range

The ranges of keys are not necessarily evenly spaced. Within each partition, we can keep keys in sorted order, so that range scans are easy. **However**, this partition can lead to hot spots. The choice of the key can largely affect how reads/writes is directed.

## 2. Partitioning by Hash of Key

A good hash func takes skewed data and makes it uniformly distributed. **However**, with hash, we lose the ability to do efficient range queries. Keys that were adjacent [[#1. Partitioning by key range]] are now scattered across all the partitions. Eg. Cassandra achieves a compromise between two strategies, a table can be declared with a `compound primary key` with several columns. Only the first part of the key sis hashed to determine the partition, the other columns are used as concatenated index for sorting the data in its SSTables.
Still in the extreme case where all reads and writes are for the same key, e.g. in social media site, a celebrity use with lots of followers, can cause a storm of activity. In this case, it is better to be handled by the application and there are tradeoffs between read and writes.

- [ ]  Read more about how twitter handles this, i remember there was an article about this.

## 3. Partitioning and Secondary Indexes

Now, we have secondary indexes (A secondary index usually doesn't identify a record uniquely but a way of searching). The problem with secondary indexes is that they don't map neatly to partitions. Solutions:

### 3.a Partitioning Secondary Indexes by Document (also known as local index)

Each partition maintains its own secondary indexes, covering **only** the documents in that partition.

1. Writes: you only need to deal with the partition that contains the document DI that you are writing.
2. Read: you need to send the query to **all** partitions and combine all the results (known as scatter/gather). The perf is prone to tail latency amplification.

### 3.b Partitioning Secondary Indexes by Term

Rather than each partition having its own secondary index, we can construct a **global index** that covers data in all partitions. However, the `global index` must also be partitioned. It is called `term-partition` since the term we are looking for determines the partition of the index.

1. Writes: slower and more complicated, since a write to a single doc now may affect multiple partitions of the index. (a term in the doc might be on a different partition on a different node, this would require a distributed transaction across all partitions affected by a write, but it is not supported in all dbs)
2. Reads: much faster than [[#3.a Partitioning Secondary Indexes by Document (also known as local index)]]
Global index update is usually `async`

# Rebalancing Partitions

Requirements for rebalancing:

1. after rebalancing, the load should be shared fairly between nodes
2. while rebalancing, the database should continue accepting reads and writes
3. No more data than necessary should be moved between nodes to make rebalancing fast and minimize network, disk I/O load etc
Ways are:

## 1. How not to do it: hash mod N

When partitioning by the hash of a key, it is best to divide the possible hashes into ranges and assign each range to a partition (in the consistent hash, there are many virtual nodes). This is to avoid rebalancing when `N` is changed.

## 2. Fixed number of partitions

Create many more partitions than there are nodes, assign several partitions to each node. When a node is added, the node can `steal` a few partitions from every existing node until partitions are fairly distributed. Thus, the number of partitions does not change, nor does the assignment of keys to partitions. The only change is the assignment of partitions to nodes. (**NOTE**:  this is used by Riak, Elasticsearch, Couchbase, Voldemort). Be extra careful about setting the initial partition number, factoring the future growth. But each partition has management overhead, thus it cannot be too large :(.

## 3. Dynamic partitioning

For [[#1. Partitioning by key range]], a fixed number of partitions with fixed boundaries would be very inconvenient, if the boundaries is got wrong. Reconfiguring the partition boundaries manually is very tedious.  This can be auto-handled, e.g. splitting a large partition into two, or merging two.

The number of partitions is proportional to the size of the dataset.

## 4. Partitioning proportionally to nodes

Make the number of partitions proportional to the number of nodes, i.e. have a fixed number of partitions per node.

# Request Routing

i.e. Service Discovery. How does the component( can be client, routing layer etc) making the routing decision? Many distributed data systems rely on a separate coordination service like ZooKeeper, which maintains the authoritative mapping of partitions to nodes,