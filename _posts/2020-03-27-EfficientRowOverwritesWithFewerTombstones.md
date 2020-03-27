---
title: Efficient Row Overwrites with Fewer Tombstones
layout: post
comments: true
tags:
 - Cassandra
---

There was an interesting “puzzle” that came up the other day that seemed worthy of writing down.

Here’s the scenario. The user will overwrite full rows of data but might only provide some of the 
columns for the row. In this scenario, they would like the empty cells to be deleted. For discussion, 
let’s consider a table with 40 columns and any given insert would have between 10 and 20 NULL fields.

The problem is that the straight-forward approach would be to insert NULL values for the empty 
columns and tell the driver to not ignore the NULLs but actually insert them. That is definitely 
simple, but will generate a lot of cell tombstones, which can cause trouble.

So, the challenge is, how do we accomplish this without generating all those troublesome tombstones?

<!--excerpt-->

## Multiple operations in a single mutation
The atomic unit in Cassandra is a mutation. However, a mutation is not just a single update to a single 
table in Cassandra. It is a collection of operations that can be done together - they’re “compatible”. 
But what does “compatible” mean?

Two updates are “compatible” if they are updates to tables in the same keyspace and are to the same 
token. That last part is a little strange. It has to have the same hash value. One easy way for that 
is if the partition key is the same type and represents the same conceptual thing. For example, 
multiple tables each with the partition key of the user ID (e.g., user_purchases and user_info). 
However, it could work for other things… but making those match on purpose is hard (like real hard…).

The reason why we need the tables to be in the same keyspace is that the keyspace defines the 
replication, and thus which nodes need to get the mutation. Therefore, if two updates have the same 
token but are in different keyspaces, then that mutation would (potentially) need to get to two 
different sets of nodes/replicas.

One interesting note is that the individual updates can have different writetime’s and TTLs.

In our case, we aren’t interested in multiple tables, but want to write two updates to the same table 
as one atomic operation.

## Row overwrite idea

We can reduce the number of tombstones by replacing the 10-20 cell tombstones that would result from inserting a row with 10-20 NULL values. One way to do this is to insert a partition tombstone, which would nullify all columns, and then insert the values and instruct the driver to ignore NULL values (or just not write those columns).

We could do two operations, first write a partition tombstone and then write the row of interest. But then we run into a small issue of which to write first and what to do if the first write fails. We have to have the partition tombstone go first, so the problem boils down to if we write the partition tombstone and then the regular write fails then we have just deleted all the data for that partition and we get no results, not even the old data. We want to avoid this.

So, how can we get these two updates (the partition delete and the insert) to happen atomically? Well, we exploit the previous observation above.

We need to handle two things. First, how are we going to submit these two updates together? Well, one way that Cassandra does that is use batches. If we send a batch with two updates to the same table (this is sufficient, but not necessary - only having the same keyspace and same token is necessary) to the coordinator, the coordinator will combine those into a single mutation and it will progress through the system as an atomic unit.

"Which kind of batch?" you may ask, logged or unlogged. Well, it actually doesn’t matter in this case. Either way the coordinator will recognize this optimization. So, we may as well use an unlogged batch.

The second question is how to make sure that the partition tombstone happens before the insert. If we send in both operations with the same timestamp, then Cassandra will do one further optimization and recognize the delete is happening and just keep that. So, how can we get the partition tombstone to "happen first"? We can use custom timestamps.

We can perform the partition delete with a timestamp of N, and do the INSERT with a timestamp of N+1 (or really N+M for any value of M>0… but really, why not 1?).

## Putting it all together

Let’s say that we would like to perform the following operation:
```sql
INSERT INTO ks.tbl(pkey, ccol, x, y, z) VALUES (1, 2, 3, NULL, NULL) WITH TIMESTAMP 123456;
```

The timestamp of 123456 is just for illustrative purposes. We would use the appropriate value for the actual timestamp. Instead, we would do the following:
```sql
BEGIN BATCH
DELETE FROM ks.tbl WHERE pkey = 1 WITH TIMESTAMP 123455;
INSERT INTO ks.tbl(pkey, ccol, x) VALUES (1, 2, 3) WITH TIMESTAMP 123456;
APPLY BATCH;
```

This will not eliminate the "tombstone problem". Tombstones will still be counted as they are encountered, and if there are too many it will Warn and then Error. But we are essentially replacing 10-20 cell tombstones with 1 partition tombstone, which will alleviate some tombstone pressure.
