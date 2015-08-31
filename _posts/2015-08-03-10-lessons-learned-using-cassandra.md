---
layout:     post
title:      "10 Lessons Learned Using Cassandra"
date:       2015-08-03 08:38:00
summary:    "How I learned to optimize for read performance in Cassandra 2.1.6."
categories: cassandra
---

I’ve grown accustomed with Cassandra as a highly available and partitionable database. Accordingly, I use it a lot at work for read-optimized queries, real-time row updates, periodic bulk table inserts, and cross-datacenter replication to support a real-time fraud engine. Personally, I found Cassandra great at those tasks, but I had to learn many lessons about Cassandra before I reached a level of satisfaction, so to save others the trouble, I will be explaining in the following sections some lessons I learned about optimizing Cassandra 2.1.6, especially for read-performance.

Aside, if anyone reading this article is new to Cassandra, I’ll provide a simple explanation of what it is and its architecture.

## Introduction to Apache Cassandra

As a NoSQL database, Cassandra is a column-oriented data store designed to be distributed, decentralized, fault tolerant, eventually consistent, and linearly scalable. In relation to the CAP theorem, Cassandra favours availability and partitioning in its core design. 

Cassandra’s most prominent architectural feature is its concept of a “ring”:  a cluster of nodes. A ring allows Cassandra to be a distributed data store which is also decentralized as it utilizes the gossip protocol to converge on the state of the cluster. 

For high availability, Cassandra utilizes replication as a first-class concept which introduces fault tolerance as data are replicated across several nodes within the ring, yet Cassandra is eventually consistent as writes requires new state to be propagated across the cluster without a centralized lock to achieve strong consistency. 

Finally, the official website for Apache Cassandra can be located at [http://cassandra.apache.org/](http://cassandra.apache.org/) for more information.

## 1. Separate the disks for commit logs from data files.

For Cassandra, the commit logs and the data files are utilized very differently as commit logs are purely append-only write operations while data files are random access.

## 2. Never use Open JDK Java Runtime Environment. Use Oracle JRE, preferably, Oracle Java 7.

Cassandra is very sensitive of the JVM from which it is running. Not using Oracle JRE can cause garbage collection performance issues in Cassandra’s periodic jobs like compaction which can actually crash a node.

## 3. Install Java Native Access if rows are cached in off-heap memory and for general performance improvements.

JNA is preferred to be installed, as Cassandra cannot efficiently utilize more than 8GB of RAM for its JVM dues to inefficiency in garbage collection for Oracle Java 7 at that limit. Furthermore, JNA improves the performance of inter-thread communications.
   
## 4. Choose an appropriate snitch for Cassandra to discover the topology of a cluster.

The prefered snitch is the `GossipingPropertyFileSnitch` which propagates information about the node’s location without requiring explicit configuration of its own location; thus, it is useful for scaling out easily.

There are snitches written specifically for Amazon AWS EC2 instances such as the `EC2Snitch` and the `EC2MultiRegionSnitch`, so research them if you use AWS a lot.

## 5. Choose the appropriate compaction strategy for the corresponding I/O pattern.

If rows are appended to an SSTable without requiring any mutation, then `SizeTieredCompactionStrategy` is appropriate.

If rows are updated often and read heavily, the `LeveledCompactionStrategy` is appropriate.

## 6. Consider row cache and key cache if rows do not belong to large tables.

If JNA available, rows can be cached in off-heap memory without impacting the JVM’s garbage collector. Nonetheless, caching rows should be very conservative and driven by a performance requirement to read a small set of data fast.

## 7. Enable compression to optimize for read performance.

Ensure the proper compression algorithm is utilized depending on a balance of I/O and CPU required due to the algorithm. The following lists the compression strategies in descending performance: `LZ4Compressor`, `SnappyCompressor`, and `DeflateCompressor`.

## 8. Consider the consistency requirements for read operations on certain tables and tune the bloom filter accordingly to allow a certain ratio of false positives.

A bloom filter is a probabilistic data structure which populates a bit array of a fixed length with a set number of hashing functions to efficiently indicate whether a certain key exists for this filter. As a result, for a given SSTable, the effectiveness of the bloom filter decreases inversely to the size of the SSTable as the array becomes dense with values.

## 9. Lower the `gc_grace_seconds` for tables which are frequently updated from the default of 10 days to something lower.

The `gc_grace_seconds` property of a table indicates how frequently should invalid data be deleted as SSTables are immutable and deleted rows are actually marked with a tombstone to be later cleaned. Having too long of a `gc_grace_seconds` property can waste disk space and require a long compaction time to clean invalid data.

## 10. Design table schemas to utilize a primary key which compliments the kinds of queries are required from the client.

Cassandra achieves great read and write performance as it does not index any column which is not a part of the primary key. Accordingly, each schema should be a specific view of data which is required.

## Conclusion

The respective points were what I learned from utilizing Cassandra, and I hope any of the advice can save anyone some headaches in the future. I highly recommend *Mastering Apache Cassandra, Second Edition* from Amazon located at [http://www.amazon.com/gp/product/1784392618/](http://www.amazon.com/gp/product/1784392618/). It is a great book which explains important concepts in Cassandra and plenty of information about optimizing and operating Cassandra.
