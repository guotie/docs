---
title: SQL Layer
summary: 
toc: false
---

The SQL Layer of CockroachDB's architecture exposes its SQL API to developers, and converts these statements into key-value operations used by the rest of the database.

{{site.data.alerts.callout_info}}If you haven't already, we recommend reading the <a href="overview.html">Architecture Overivew</a>.{{site.data.alerts.end}}

<div id="toc"></div>

## Overview

Once CockroachDB has been deployed, developers need nothing more than a connection string to the cluster and SQL statements to start working.

Because CockroachDB's nodes all behave symmetrically, developers can send requests to any node (which means CockroachDB works well with load balancers). Whichever node receives the request acts as the "gateway node," as other layers process the request.

When developers send requests to the cluster, they arrive as SQL statements, but data is ultimately written to and read from the storage layer as key-value (KV) pairs. To handle this, the SQL layer converts SQL statements into a plan of KV operations, which it passes along to the Transaction Layer.

### Interactions with Other Layers

In relationship to other layers in CockroachDB, the SQL Layer:

- Sends requests to the Transaction Layer.

## Components

### Relational Structure

Developers experience data stored in CockroachDB in a relational structure, i.e., rows and columns. Sets of rows and columns are organized into tables. Collections of tables are organized into databases. Your cluster can contain many databases.

Because of this structure, CockroachDB provides typical relational features like constraints (e.g., foreign keys). This lets application developers trust that the database will ensure consistent structuring of the application's data; data validation doesn't need to be built into the application logic separately.

### SQL API

CockroachDB implements a large portion of the ANSI SQL standard to manifest its relational structure. You can view [all of the SQL features CockroachDB supports here](../sql-feature-support.html).

Importantly, through the SQL API, we also let developers use ACID-semantic transactions just like they would through any SQL database (`BEGIN`, `END`, `ISOLATION LEVELS`, etc.)

### PostgreSQL Wire Protocol

SQL queries reach your cluster through the PostgreSQL wire protocol. This makes connecting your application to the cluster simple by supporting most PostgreSQL-compatible drivers, as well as many PostgreSQL ORMs, such as [GORM (Go)](../build-a-go-app-with-cockroachdb-gorm.html) and [Hibernate (Java)](../build-a-java-app-with-cockroachdb-hibernate.html).

### SQL Parser, Planner, Executor

After your node ultimately receives a SQL request from a client, CockroachDB parses the statement, creates a query plan, and then executes the plan.

#### Parsing

Received queries are parsed against our `yacc` file (which describes our supported syntax), and converts the string version of each query into an [Abstract Syntax Tree](https://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST).

#### Planning

CockroachDB then uses the AST version of the query to generate a tree of `planNodes`. Each `planNode` contains the  CRUD-type operations from the SQL statement in a way that lower levels of CockroachDB can understand, i.e., as key-value operations. You can see the `planNodes` a query generates using [`EXPLAIN`](../explain.html).

Planning also includes steps to analyze the SQL statements against the expected AST expressions, including operations like type checking.

#### Executing

The key-value operations in the `planNodes` are then executed, which begins by communicating with the Transaction Layer.

As operations execute, their values are also encoded. On the opposite side, as the results of queries return, they are decoded in this step.

### Encoding

Because lower layers of CockroachDB deal primarily in bytes, SQL queries must have values (originally represented as strings when sent by the client) encoded, i.e., converted into bytes. And, on the other side, bytes in response to queries must be decoded into strings which can be returned to the client.

To achieve this, CockroachDB uses two types of encoding:

- **Order-preserving** for indexed columns, which maintain the same sort order as the data it represents
- **Value encoding** for *non*-indexed columns, which *does not* maintain its order but *does* consume less space

You can find more exhaustive detail in the [Encoding Tech Note](https://github.com/cockroachdb/cockroach/blob/master/docs/tech-notes/encoding.md).

### DistSQL

Because CockroachDB is a distributed database, we've developed a Distributed SQL (DistSQL) optimization tool for some queries, which can dramatically speed up queries that involve many ranges. Though DistSQL's architecture is worthy of its own documentation, this cursory explanation can provide some insight into how it works.

In non-distributed queries, the coordinating node receives all of the rows that match its query, and then performs any computations on the entire data set. 

However, for DistSQL-compatible queries, each node does computations on the rows it contains, and then sends the results (instead of the entire rows) to the coordinating node. The coordinating node then aggregates the results from each node, and finally returns a single response to the client.

This dramatically reduces the amount of data brought to the coordinating node, and leverages the well-proven concept of parallel computing, ultimately reducing the time it takes for complex queries to complete. In addition, this processes data on the node that already stores it, which lets CockroachDB handle row-sets that are larger than an individual node's storage.

To run SQL statements in a distributed fashion, we introduce a couple of concepts:

- **Logical plan**: Similar to the AST/`planNode` tree described above, it represents the abstract (non-distributed) data flow through computation stages.
- **Physical plan**: A physical plan is conceptually a mapping of the logical plan nodes to physical machines running `cockroach`. Logical plan nodes are replicated and specialized depending on the cluster topology. Like `planNodes` above, these components of the physical plan are scheduled and run on the cluster.

You can find much greater detail in the [DistSQL RFC](https://github.com/cockroachdb/cockroach/blob/master/docs/RFCS/20160421_distributed_sql.md).

## Technical Interactions with Other Layers

### SQL & Transaction Layer

KV operations from executed `planNodes` are sent to the Transaction Layer.

## What's Next?

Learn how CockroachDB handles concurrent requests in the [Transaction Layer](transaction-layer.html).
