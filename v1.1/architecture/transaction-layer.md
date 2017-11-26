---
title: Transaction Layer
summary: 
toc: false
---

The Transaction Layer of CockroachDB's architecture implements support for ACID transactions by coordinating concurrent operations.

{{site.data.alerts.callout_info}}If you haven't already, we recommend reading the <a href="overview.html">Architecture Overivew</a>.{{site.data.alerts.end}}

<div id="toc"></div>

## Overview

Above all else, CockroachDB believes consistency is the most important feature of a database––without it, developers cannot build reliable tools, and businesses suffer from potentially subtle and hard-to-detect anomalies.

To provide consistency, CockroachDB implements full support for ACID transaction semantics in the Transaction Layer. However, it's important to realize that *all* statements are handled as transactions, including single statements––this is sometimes referred to as "autocommit mode" because it behaves as if every statement is followed by a `COMMIT`.

For code samples of using transactions in CockroachDB, see our documentation on [transactions](../transactions.html#sql-statements).

Because CockroachDB enables transactions that can span your entire cluster (including cross-range and cross-table transactions), it optimizes correctness through a somewhat-pessimistic concurrency control system.

### Concurrency Control

#### Writing

When the Transaction Layer executes write operations, it doesn't directly write values to disk. Instead, it creates two things that help it mediate a distributed transaction:

- A **Transaction Record** stored in the range where the first write occurs, which includes:
	- The transaction's current state (which starts as `PENDING`, and ends as either `COMMITTED` or `ABORTED`).
	- A list of all writes related to the transaction (known as Write Intents).
    It's also important to realize that this Transaction Record is also a KV entry, so it benefits from [replication](replication-layer.html), ultimately letting a transaction continue even if some nodes involved in the transaction's writes/reads fail.
- **Write Intents** for all of a transaction’s writes, which represent a provisional, uncommitted state. These are essentially the same as standard [multi-version concurrency control (MVCC)](storage-layer.html#mvcc) values but also contain a pointer to its parent Transaction Record.

To perform a write, CockroachDB also checks for:
- Newer MVCC values, in which case the transaction is restarted
- Other Write Intents for the same keys, which are resolved as a [transaction conflict](#transaction-conflicts)
- If a read for the key has ever occurred at a higher timestamp (through the [Timestamp Cache](#timestamp-cache)), in which case the transaction is restarted

#### Reading

Reads return the most-recent MVCC value with a timestamp less than its own; if that value is a Write Intent, the read must be resolved as a [transaction conflict](#transaction-conflicts). 

Notably, this mechanism also enables us to offer [`AS OF SYSTEM TIME` support](https://www.cockroachlabs.com/blog/time-travel-queries-select-witty_subtitle-the_future/).

#### Commits

When the Transaction Layer processes the request to `COMMIT` the transaction, the coordinating node who received the SQL request proceeds through the following checks:

1. If the Transaction Record's status is `ABORTED`, the SQL Layer becomes aware of the issue and makes decisions from there, either restarting the transaction or notifying the client of the failure.

2. If any of the transaction's operations are in the `PushTxnQueue`, the `COMMIT` is deferred until the operation's resolved.

3. If the Transaction Record's status is `PENDING`, it's moved to `COMMITTED` and the client receives an acknowledgment that the transaction succeeded. At this point, the client is free to begin sending more requests to the cluster.

#### Cleanup

After the transaction has been resolved, CockroachDB [resolves all Write Intents](#resolving-write-intent) listed in the Transaction Record.

This cleanup is simply an optimization, though. Whenever an operation encounters a Write Intent, it checks its Transaction Record; if it's in any status besides `PENDING`, the operation can perform the same resolution.

### Interactions with Other Layers

In relationship to other layers in CockroachDB, the Transaction Layer:

- Receives KV operations from the SQL Layer.
- Controls the flow of KV operations sent to the Distribution Layer.

## Technical Details & Components

### Gateway Node

When a client sends a transaction to the SQL API, it works as the gateway node for the entire transaction, performing a number of record-keeping tasks:

- Creating the Transaction Record on the first range the transaction operates on
- Choosing a timestamp for the entire transaction
- Using `TxnCoordSender` to maintain the transaction
- Responding to the client

Though CockroachDB is a distributed, highly available database, if a the gateway node dies during a transaction, the client will have to re-send the transaction to the cluster.

### Time & Hybrid Logical Clocks

To help rationalize time in a distributed environment––which must account for discrepancies between multiple nodes' clocks––CockroachDB implements hybrid-logical clocks (HLC) which are composed of a physical component (thought of as and always close to local wall time) and a logical component (used to distinguish between events with the same physical component). For much greater detail, check out the [HLC paper](http://www.cse.buffalo.edu/tech-reports/2014-04.pdf).

When initiating a transaction, the Gateway Node chooses a timestamp for the entire transaction using HLC time. Whenever a transaction's timestamp is mentioned, it's an HLC value. This timestamp is used to both track versions of values (through [multiversion concurrency control](storage-layer.html#mvcc), as well as provide our transactional isolation guarantees.

This timestamp is included whenever operations are sent to other nodes. When nodes receive requests, they inform their local HLC of the request's timestamp, which enforces distributed consistency by guaranteeing that all data that is read or overwritten is at a timestamp less than the next HLC time (i.e. is "in the past").

CockroachDB leverages HLC timestamp ordering to optimize read performance, as well. By letting the node primarily responsible for a range (known as its [Leaseholder](replication-layer.html#leases)) serve reads whenever the read's timestamp is greater than the MVCC value it's reading (i.e., the read always happens "after" the write), we can ensure consistency without unnecessarily forcing reads to achieve consensus.

#### Max Clock Offset Enforcement

To ensure correctness among distributed nodes, you can identify a Maximum Clock Offset. Because CockroachDB relies on clock synchronization, nodes run a version of [Marzullo’s algorithm](http://infolab.stanford.edu/pub/cstr/reports/csl/tr/83/247/CSL-TR-83-247.pdf) amongst themselves to measure maximum clock offset within the cluster. If a node's timestamp exceeds the maximum offset, it commits suicide to prevent any potential inconsistencies within the cluster.

For more detail about the risks that large clock offsets can cause, see [What happens when node clocks are not properly synchronized?](../operational-faqs.html#what-happens-when-node-clocks-are-not-properly-synchronized)

### Timestamp Cache

To provide serializability, whenever an operation reads a value, we store the operation's timestamp in a Timestamp Cache, which shows the high-water mark for values being read.

Whenever a write occurs, its timestamp is checked against the Timestamp Cache. If the timestamp is less than the Timestamp Cache's latest value, we attempt to move the timestamp for its transaction forward to a later time. In the case of serializable transactions, this causes them to restart in the second phase of the transaction.

### client.Txn and TxnCoordSender

In the [SQL Layer](sql-layer.html), CockroachDB converts all SQL statements into key-value (KV) operations, which is how data is ultimately stored and accessed. 

All of the KV operations generated from the SQL layer use `client.Txn`, which is the transactional interface for the CockroachDB KV layer (as we discussed above, all statements are treated as transactions, so all statements use this interface).

However, `client.Txn` is actually just a wrapper around `TxnCoordSender`, which plays a crucial role in our code base by:

- Dealing with transactions' state. After a transaction is started, `TxnCoordSender` starts asynchronously sending heartbeat messages to that transaction's Transaction Record, which signals that it should be kept alive. If the `TxnCoordSender`'s heartbeating stops, the Transaction Record is moved to the `ABORTED` status.
- Tracking each written key or key range as Transaction Records over the course of the transaction. 
- Resolving the transaction's Write Intents once it's committed or aborted. All of the transaction's requests go through the Gateway Node's `TxnCoordSender` to account for all of its Write Intents to optimizes this cleanup process.

After setting up this bookkeeping, the request is passed to the `DistSender` in the Distribution Layer.

### Transaction Records

When a transaction starts, `TxnCoordSender` writes a Transaction Record to the range containing the first key modified in the transaction. As mentioned above, the Transaction Record provides the system with a source of truth about the status of a transaction.

The Transaction Record expresses one of the following dispositions of a transaction:

- `PENDING`: The initial status of all values, indicating that the Write Intent's transaction is still in progress.
- `COMMITTED`: Once a transaction has completed, this status indicates that the value can be read.
- `ABORTED`: If a transaction fails or is aborted by the client, it's moved into this state.

The Transaction Record for a committed transaction remains until all its Write Intents are converted to MVCC values. For an aborted transaction, the Transaction Record can be deleted at any time, which also means that CockroachDB treats missing Transaction Records as if they belong to aborted transactions.

### Write Intents

Values in CockroachDB are not directly written to the storage layer; instead everything is written in a provisional state known as a "Write Intent." These are essentially multi-version concurrency control values (also known as MVCC, which is explained in greater depth in the Storage Layer) with an additional value added to them which identifies the Transaction Record to which the value belongs.

Whenever an operation encounters a Write Intent (instead of an MVCC value), it looks up the status of the the Transaction Record to understand how it should treat the Write Intent value.

#### Resolving Write Intent

Whenever an operation encounters a Write Intent for a key, it attempts to "resolve" it, the result of which depends on the Write Intent's Transaction Record:

- `COMMITTED`: The operation reads the Write Intent and converts it to an MVCC value by removing the Write Intent's pointer to the Transaction Record.
- `ABORTED`: The Write Intent is ignored and deleted.
- `PENDING`: This signals there is a [transaction conflict](#transaction-conflicts), which must be resolved.

### Isolation Levels

Isolation is an element of [ACID transactions](https://en.wikipedia.org/wiki/ACID), which determines how concurrency is controlled, and ultimately guarantees consistency.

Because CockroachDB aims to provide a highly consistent database, it only offers two isolation levels:

- **Serializable Snapshot Isolation** _(Serializable)_ transactions are CockroachDB's default (equivalent to ANSI SQL's `SERIALIZABLE` isolation level, which is the highest of the four standard levels). This isolation level does not allow any anomalies in your data, which is largely enforced by refusing to move the transaction's timestamp, or by aborting the transaction if its timestamp is moved. This enforces serializability in your data.

- **Snapshot Isolation** _(Snapshot)_ transactions trade correctness for improvements in performance for high-contention work loads. This is achieved by allowing the transaction's timestamp to be moved during [transaction conflicts](#transaction-conflicts), which allows an anomaly called [write skew](https://en.wikipedia.org/wiki/Snapshot_isolation) to occur.

### Transaction Conflicts

CockroachDB's transactions allow the following types of conflicts:

- **Write/Write**, where two `PENDING` transactions create Write Intents for the same key.
- **Write/Read**, when a read encounters an existing Write Intent with a timestamp less than its own.
There's also read/write, in which a write discovers that a read has already been served after its proposed timestamp. This pushes the write transaction's timestamp and will cause a transaction restart error at some point if the transaction is serializable (the restart error will currently happen when the transaction attempts to commit, although we may change this so the error happens immediately in some cases). The pushTxnQueue is not involved for read/write conflicts; this comes from the timestamp cache.

To make this simpler to understand, we'll call the first transaction `TxnA` and the transaction that encounters its Write Intents `TxnB`.

CockroachDB proceeds through the following steps until the first of the following conditions is met:

1. If either transaction has an explicit priority set (i.e. `HIGH`, or `LOW`), the transaction with the lower priority has its Transaction Record's status changed to `ABORTED`.

2. `TxnB` tries to push `TxnA`'s timestamp forward.
    
    This succeeds only in the case that `TxnA` has snapshot isolation and `TxnB`'s operation is a read. In this case, the [write skew](https://en.wikipedia.org/wiki/Snapshot_isolation) anomaly occurs.

3. `TxnB` enters the `PushTxnQueue` to wait for `TxnA` to complete.

### PushTxnQueue

The `PushTxnQueue` tracks all transactions that could not push a transaction whose Write Intents they encountered, and must wait for the blocking transaction to complete before they can proceed. 

The `PushTxnQueue`'s structure is a map of blocking transaction IDs to those they're blocking. For example:

~~~
txnA -> txn1, txn2
txnB -> txn3, txn4, txn5
~~~

Importantly, all of this activity happens on a single node, which is the leader of the range's Raft group that contains the Transaction Record.

Once the transaction does resolve––by committing or aborting––a signal is sent to the `PushTxnQueue`, which lets all transactions that were blocked by the resolved transaction begin executing.

Blocked transactions also check the status of their own transaction periodically to ensure its still active. If the blocked transaction was aborted, it's simply removed.

If there is a deadlock between transactions (i.e., they're each blocked by each other's Write Intents), one of the transactions is randomly aborted.

## Technical Interactions with Other Layers

### Transaction & SQL Layer

The Transaction Layer receives KV operations from `planNodes` executed in the SQL Layer, and returns KV data and statuses.

### Transaction & Distribution Layer

The `TxnCoordSender` sends its KV requests to `DistSender` in the Distribution Layer, as well as receives its responses, which are ultimately what gets passed back to the SQL Layer.

## What's Next?

Learn how CockroachDB presents a unified view of your cluster's data in the [Distribution Layer](distribution-layer.html).
