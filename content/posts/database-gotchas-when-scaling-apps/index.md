+++
title = '3 database gotchas when building apps for scale'
description = "Database issues that arise when scaling applications"
image = "database-thumbnail.png"
date = 2024-05-29
categories = ["Database Administration"]
tags = ["SQL Performance", "SQL Developer", "Database Scaling", "MySQL"]
draft = false
+++

- [Excessive database locks](#excessive-database-locks)
- [Read-after-write consistency](#read-after-write-consistency)
- [Index limitations](#index-limitations)

When building an application, the database is often an afterthought. The database used in a development environment
often contains limited data with little traffic. However, when the application is deployed to production, real-world
traffic can expose issues that were not caught in development or testing.

In this article, we cover issues we ran into with our customers. We assume the production application is deployed with
one master and one or more read replicas. See this article on
[creating a MySQL slave replica in dev environment](../mysql-master-slave-replication).

## Excessive database locks {#excessive-database-locks}

One write query can bring your database to its knees if it locks too many rows.

Consider this simplified INSERT with a subquery transaction:

```sql
INSERT INTO software_counts (host_id, count)
SELECT host_id, COUNT(*) as count FROM host_software
GROUP BY host_software.host_id;
```

{{< figure src="simple-insert-with-subquery.svg" title="Simplified INSERT with a subquery" alt="Simplified INSERT with a subquery" >}}

The above query scans the entire `host_software` table index to create a count. While the database is doing the scan and
the INSERT, it locks the `host_software` table, preventing other transactions from writing to that table. If the table
and insert are large, the query can hold the lock for a long time. In production, we saw a lock time of over 30 seconds,
creating a bottleneck and spiking DB resource usage.

Pay special attention to the following queries, as they can cause performance issues:

- `COUNT(*)`
- Using a non-indexed column, like `WHERE non_indexed_column = value`
- Returning a large number of rows, like `SELECT * FROM table`

One way to solve the above performance issue is to separate the `SELECT` and `INSERT` queries. First, run the `SELECT`
query on the replica to get the data, then run the `INSERT` query on the master to insert the data. We completely
eliminate the lock since the read is done on the replica. This article goes through
[a specific example of optimizing an INSERT with subqueries](.../mysql-query-performance-insert-subqueries).

As general advice, avoid running `SELECT` queries and subqueries on the master, especially if they scan the entire
table.

## Read-after-write consistency {#read-after-write-consistency}

When you write to the master and read from the replica, you might not see the data you wrote. The replica is not in sync
with the master in real time. In our production, the replica is usually less than 30 milliseconds behind the master.

{{< figure src="read-after-write-consistency.svg" title="Read-after-write database issue" alt="Read-after-write database issue" >}}

These issues are typically not caught in development since dev environments usually have one database instance. Unit or
integration tests might not even see these issues if they run on a single database instance. Even in testing or small
production environments, you might only see these issues if the replica sync time is high. Customers with large
deployments may be experiencing these consistency issues without the development team knowing about it.

One way to solve this issue is to read from the master after writing to it. This way, you are guaranteed to see the data
you just wrote. In
[our Go backend](https://github.com/fleetdm/fleet/blob/b7aac2cfabf17fcb5142808fb80352113710ec5c/server/contexts/ctxdb/ctxdb.go#L17),
forcing reads from the master can be done by updating the `Context`:

```go
ctxUsePrimary := ctxdb.RequirePrimary(ctx, true)
```

However, additional master reads increase the load on the master, defeating the purpose of having a replica for read
scaling.

In addition, what about expensive read queries, like `COUNT(*)` and calculations, which we don't want to run on the
master? In this case, we can wait for the replica to catch up with the master.

One generic approach to waiting for the replica is to read the last written data from the replica and retry the read if
the data is not found. The app could check the `updated_at` column to see if the data is recent. If the data is not
found, the app can sleep for a few milliseconds and retry the read. This approach is imperfect but a good compromise
between read consistency and performance.

Note:
[The default precision of MySQL date and time data types is 1 second (0 fractional seconds)](https://dev.mysql.com/doc/refman/8.0/en/date-and-time-type-syntax.html#:~:text=default%20precision%20is%200.).

## Index limitations {#index-limitations}

### What are SQL indexes?

Indexes are a way to optimize read queries. They are a data structure that improves the speed of data retrieval
operations on a database table at the cost of additional writes and storage space to maintain the index data structure.
Indexes are created using one or more database columns and are stored and sorted using a B-tree or a similar data
structure. The goal is to reduce the number of data comparisons needed to find the data.

{{< figure src="database-index.svg" title="Database index" alt="Database index" >}}

Indexes are generally beneficial. They speed up read queries but slightly slow down write queries. Indexes can also be
large and take up a lot of disk space.

### Index size is limited

As the product grows with more features, the number of columns in a specific table can also increase. Sometimes, the new
columns need to be part of a unique index. However, the
[maximum index size in MySQL is 3072 bytes](https://dev.mysql.com/doc/refman/8.0/en/innodb-limits.html). This limit can
be quickly reached if columns are of type `VARCHAR` or `TEXT`.

```sql
CREATE TABLE `activities` (
  `user_name` VARCHAR(255) NOT NULL,
```

One way to solve the issue of hitting the index size limit is to create a new column that makes the hash of the other
relevant column(s), and use that as the unique index. For example, in our backend
[we use a `checksum` column in the `software` table to create a unique index for a software item](https://github.com/fleetdm/fleet/blob/6f008b40f24bcd000c1450d7438be99d30c518c5/server/datastore/mysql/schema.sql#L1450).

### Foreign keys may cause performance issues

If a table has a foreign key, any insert, update, or delete with a constraint on the foreign key column will lock the
corresponding row in the parent table. This locking can lead to performance issues when

- the parent table is large
- the parent has many foreign key constraints
- the parent table or child tables are frequently updated

The performance issue manifests as excessive lock wait times for queries. One way to solve this issue is to remove the
foreign key constraint. Instead, the application code can handle the data integrity checks that the foreign key
constraint provides. In our application, we run a regular clean-up job to remove orphaned child rows.

## Bonus database gotchas

Additional database gotchas that we have seen in production include:

- [Prepared statements consuming too much memory](../sql-prepared-statements-are-broken-when-scaling-applications)
- [Deadlocks caused by using an UPDATE/INSERT upsert pattern](../mysql-upsert-deadlock)

Also, we recently [solved a problem in production with distributed lock](../distributed-lock).

## 3 database gotchas video

{{< youtube N-wzNq-sEwo >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
