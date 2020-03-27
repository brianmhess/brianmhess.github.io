---
title: Change Data Capture in Apache Cassandra
layout: post
comments: true
tags:
 - Cassandra
---

A common data flow we see is people wanting their application to write to Cassandra as the database of record and then have downstream systems consume the changes once they’ve been written to Cassandra. This is a common feature in database systems, typically called Change Data Capture (CDC), and a common flow for enterprise data layers.

Apache Cassandra introduced Change Data Capture in version 3.8, via [CASSANDRA-8844](https://issues.apache.org/jira/browse/CASSANDRA-8844). A lot of people are interested in this feature to satisfy the desired data flow, so let me try to explain how CDC works in Cassandra.

<!--excerpt-->

## The Goal

The idea behind the CDC feature in Cassandra is to capture all mutations (INSERT, UPDATE, DELETE) and make them available for another process to process these files, and then delete them. There is no requirement that CDC be designed at the coordinator or at the replica, just that writes that got into Cassandra are available to be processed for other purposes.

A few cardinal rules:
* all writes that are successfully in Cassandra make it into the CDC stream
* no writes are in the CDC stream that are not also in Cassandra
* all write operations (INSERT, UPDATE, DELETE (row, range, partition)) must be captured
* the solution needs to be tolerant of node failures (and the node never coming back) and network partitions

## The Coordinator or the Replica

When thinking about where we can tee off a feed of the writes coming into the cluster, there are really two places we can tap in: at the coordinator, or at the replica.

### The Coordinator

The coordinator is a nice place to think about teeing off a change feed. It is a single point so we don’t have to bring in the distributed (and replicated) nature of Cassandra’s architecture.

The main problem with a coordinator-based approach is that the coordinator isn’t a definitive authority on whether a write has been persisted into the database (as a whole) or not. That is, if a query times out (e.g., if all three replicas take too long to respond to a QUORUM/LOCAL_QUORUM write request), the coordinator cannot be sure if any of the writes were accepted by any of the replicas. If even one replica receives the write, then via anti-entropy operations (e.g., repair in open-source Cassandra or NodeSync in DataStax Enterprise (DSE)) all replicas will get the write even though the coordinator has no knowledge that the write was successful. If we depend on the coordinator publishing the changes, then it would miss the write in this scenario. So, there would be a write in Cassandra that is not in the change feed.

We could, of course, have the coordinator log the write prior to sending it to the replicas. For example, we could use the Cassandra Trigger mechanism to do this. However, if the write actually did not succeed on any replica, then there would be a write in the change feed that is not in the database.

We could create some form of two-phase commit for the writes at the coordinator where the changes are staged and when the acknowledgements from the replicas are received then the writes are “committed” to the change feed. In the case where the write timed out, a more expensive path could be taken to attempt to read the write from the replicas to see if any of them received it, and if so then promote the staged write to the change feed. This is a bit complex, but could work (though there may be some additional sharp corners to be concerned about).

Even if the coordinator could properly capture the changes, the other issue would be about the durability of the changes that a given coordinator has captured. If they are stored locally, such as how hints in 3.0+ are handled, then if a node goes offline so do the changes it logged. In order to be tolerant to unrecoverable node failure, the change feed would need to be stored on multiple nodes (such as how logged batches are stored). This greatly complicates the change log storage. It also makes it a bit difficult to consume since all changes are stored in multiple places.

### The Replica(s)

The other location to consider is to capture the changes at the replica. One obvious thing going for it is that the replica is the definitive source of whether a piece of data is in the database or not. If the replica has it, it can be returned to a query, and (obviously) if it does not have it, then it cannot be returned and therefore it is as if it does not exist.

On the replica, a given mutation is not durable until it has been persisted to disk. There are two places in the write path that satisfy this: the commitlog and the sstable itself. Recall the data flow for writes: 

1. the replica receives the write over the wire
2. the replica logs the write to the commitlog
   1. This makes the write durable. If Cassandra was to be terminated (say via kill), when Cassandra is restarted, the commitlog will be replayed and the write will again proceed as below.
3. the replica processes the write into the memtable (in RAM)
4. the replica acknowledges a successful write to the coordinator
5. whenever there is sufficient memory pressure (or some other times, like an explicit “nodetool flush” operation), some memtables are flushed to sstables on disk (freeing up RAM)
   1. when the memtable is flushed to disk, all mutations in that memtable are now durable. This means that we no longer need to keep those mutations in the commitlog, so Cassandra marks those mutations as flushed.
   2. when a commitlog has all of its mutations marked as flushed, Cassandra can delete the entire commitlog file.

One thing to note is that while #2 above is written as if the write to the commitlog is synchronous, the default in Cassandra is to write to the commitlog asynchronously, fsyncing the data to disk every 10 seconds by default (but configurable in cassandra.yaml). This means that in the case of an inelegant (or abrupt) process termination, we could lose up to 10 seconds (by default) worth of mutations on this replica. The "on this replica" part is important, because even though some mutations might be lost on this replica, Cassandra is a distributed and replicated system, and the other replicas are also receiving these writes, so in order to lose the mutation completely, all three replicas would have to go down at the same time, which is highly unlikely.

So, why go through the write path on the replica? Largely so we can point out where in that flow would be a good place to tap into the changes. The obvious choice here is the commitlog. That is the first place that makes a write durable. The other place would be in the sstable flush, however we cannot be sure how often that will happen as it is mainly driven by memory consumption (and needing more memory space for new mutations).

What we could do here is use the commitlog as the CDC log and simply keep that file around instead of deleting it when data is flushed to sstables This allows a change feed consumer to process the commitlog file and then choose when to delete the file(s).

But what about if writes come into Cassandra that don’t go through the normal write path? That can happen if data is streamed to the replica (e.g., via rebuild, bootstrap, or even sstableloader). So, if we choose a commitlog-based solution, we will also need a way to get the mutations from streaming operations into the CDC feed.

One clear challenge with a replica-based solution is that each replica operates independently, so we will end up with the mutation going into the commitlog on each replica. So, if we have the standard replication factor of 3, then each mutation will be captured three times. This is good for durability concerns (if we lose one replica, we have the mutation on the other 2 replicas). One challenge it presents is that the process that consumes the changes will receive each mutation 3 times (or whatever the replication factor is); each mutation is captured in "RF-licate".

Another complication here is that each replica will record the mutations, but they could be (and almost surely will be) in a different order in the commitlogs on the replicas. Additionally, mutations can be delayed significantly getting to a replica. It could be that a node was offline for a short time (or perhaps going through a JVM garbage collection operation) so the write would be hinted by the coordinator, delaying it by a few seconds or so. It could also be that a node receives the mutation during a repair operation (or the analogous NodeSync operation), which could be delayed up to several days (a common best practice is to set the repair schedule to every 7 days (and the default gc_grace_seconds to 10 days)).

Any deduping operation would need to tolerate data being out of order on each replica, and data potentially being delayed by up to a repair cycle on a replica.

Another place to tap into the replica-based write stream is to implement a custom secondary index. That custom secondary index could be added to a table of interest and all mutations could be stored in a secondary storage (owned by this custom secondary index) to be processed. This results in an additional storage of every mutation. It also suffers the same challenges on data order and deduplication as the commitlog approach.

## Cassandra CDC

CASSANDRA-8844 chose a replica-based approach, utilizing the commitlog as sketched above.

CDC is a feature that can be enabled on a per-table basis via CQL DDL.
```sql
ALTER TABLE ks.tbl WITH cdc = TRUE;
```

Instead of a separate change data capture log, Cassandra will mark individual mutations on a CDC-enabled table. A commitlog contains mutations across all tables in Cassandra, so in addition Cassandra will mark if a given commitlog contains any CDC mutations or not.

Once all of the mutations in a commitlog have been flushed to sstables, Cassandra is finished with the commitlog (as explained above). However, if the commitlog does contain a CDC record (or more) instead of deleting the file, Cassandra will move that file to another directory, `cdc_raw` (the default is `/var/lib/cassandra/cdc_raw`, but is configurable in cassandra.yaml). At this point, Cassandra is finished with the file, and it is the responsibility of some other process to delete these files before the disk fills up.

If the CDC logs start backing up and exceed a size threshold (settable in cassandra.yaml, but defaulting to 4GB) then any writes to a CDC-enabled table will fail with a `WriteTimeoutException`. Thus, care must be taken to process these files (and delete them) in a timely fashion.

CASSANDRA-8844 did one other clever thing. To handle the case of data coming to a replica via the streaming API (e.g., repair), Cassandra will artificially create commitlog entries for all mutations in a CDC-enabled table received this way so that they can show up in the CDC log to be processed by the CDC consumers. In this way, data in a CDC-enabled table cannot show up on a replica without being included in the CDC log.

There are a few node-level CDC configuration options in cassandra.yaml:
* `cdc_enabled` - flag to enable CDC on the node; defaults to `false`
* `cdc_raw_directory` - the location of the cdc_raw directory; defaults to `/var/lib/cassandra/cdc_raw`
* `cdc_total_space_in_mb` - the total amount of space (in MB) that the `cdc_raw` directory can grow to; defaults to `4096`
* `cdc_free_space_check_interval_ms` - once the `cdc_total_space_in_mb` has been exceeded, how often to recheck the size of the `cdc_raw` directory; defaults to `250`

### CDC Consumer

The idea is that the user will provide a CDC consumer which must be run on each replica. That CDC consumer will monitor the `cdc_raw` directory, process files as they arrive, and delete the file to make room for more files.

The CDC consumer is an implementation of the `CommitLogReader` (`org.apache.cassandra.db.commitlog.CommitLogReader`). Note, however, that the interface that the user must use to process the mutations is the Cassandra `Mutation` class (`org.apache.cassandra.db.Mutation`), which is not nearly as friendly as the Cassandra Java driver’s API.

Cassandra does not have a mechanism to manage the life cycle of a CDC consumer; that is the responsibility of the CDC consumer. That is, some mechanism needs to be responsible for starting, stopping, and handling errors of the CDC consumer (e.g., needing to restart due to some error). If the CDC consumer’s life cycle is not handled properly, the `cdc_raw` directory will fill up and Cassandra will start throwing errors on writes.

## Observations

A few observations/cautions:

* If `commitlog_sync` is set to `periodic` (the default), then mutations will not show up on disk in the commitlog until `commitlog_sync_period_in_ms`, which defaults to 10000, so mutations can take up to 10 seconds until they can be processed by a CDC consumer.
* CDC consumers consume data in the `cdc_raw` directory, not the commitlog directory. So, files are delayed getting to the CDC consumer until Cassandra is finished with the commitlog (all of the mutations in the commitlog have been flushed to sstables). There is an improvement ([CASSANDRA-12148](https://issues.apache.org/jira/browse/CASSANDRA-12148)) in Cassandra 4.0 that helps lower this delay.
* If a CDC consumer does not clean up (or keep up with) the files in the `cdc_raw` directory, then normal user write operations will be impacted. For example, it is a real problem to enable CDC but not set up a CDC consumer.
* All mutations will be in Cassandra in RF-licate, so the CDC consumers will either need to somehow handle distributed deduplication, or process/produce each mutation in RF-licate. For example, if the goal of your CDC consumer was to push the mutations to another Cassandra cluster, then the naive approach of having each CDC consumer write to the destination cluster would result in a 3-fold increase in the write workload (assuming an RF of 3) to the destination cluster over the source cluster. Assuming the data model in the destination Cassandra cluster is idempotent, Cassandra will properly dedup the data, however the write workload would be wastefully high (a trade-off that may or may not be palatable).
* The Cassandra `Mutation` interface is not as friendly as the Cassandra Java driver interface, so taking on writing a CDC consumer is not a beginner-type endeavor.
* The life cycle of a CDC consumer is the responsibility of the operator, not Cassandra. If you do not handle that with care, it can result in writes failing on CDC-enabled tables.

## Wrapping Up

Cassandra CDC is a great foundational step for being able to process a change feed coming into Cassandra. The replica-based approach allows us to achieve the primary goals:
* All changes in Cassandra are in the CDC feed
* All changes in the CDC feed are in Cassanrda
* All mutations (INSERT, UPDATE, DELETE) are supported

That said, the solution still requires some work by the operator to provide a CDC consumer:
* The CDC consumer must consume the CDC log in a timely fashion or else can negatively impact Cassandra operation.
* The CDC consumer must handle deduping itself.
* There is a delay between the mutation being in the Cassandra replica and being available for processing by the CDC consumer (up to 10 seconds to get into the commitlog, longer to be moved to cdc_raw).
* The API is a low-level API and not for the faint of heart.
