+++
title = 'Optimize MySQL query performance: INSERT with subqueries'
description = "Real world example of fixing a slow MySQL query doing an INSERT with subqueries"
image = "INSERT with subqueries.png"
date = 2024-05-06
categories = ["Database Administration"]
tags = ["SQL Developer", "MySQL", "SQL Performance"]
draft = false
+++

## Introduction

We recently encountered a performance issue in production. Once an hour, we saw a spike in average DB lock time, along with occasional deadlocks and server errors. We identified the problematic query using [Amazon RDS](https://aws.amazon.com/rds/) logs. It was an `INSERT` statement with subqueries.

```sql
INSERT INTO policy_stats (policy_id, inherited_team_id, passing_host_count, failing_host_count)
SELECT
    p.id,
    t.id AS inherited_team_id,
    (
        SELECT COUNT(*)
        FROM policy_membership pm
        INNER JOIN hosts h ON pm.host_id = h.id
        WHERE pm.policy_id = p.id AND pm.passes = true AND h.team_id = t.id
    ) AS passing_host_count,
    (
        SELECT COUNT(*)
        FROM policy_membership pm
        INNER JOIN hosts h ON pm.host_id = h.id
        WHERE pm.policy_id = p.id AND pm.passes = false AND h.team_id = t.id
    ) AS failing_host_count
FROM policies p
CROSS JOIN teams t
WHERE p.team_id IS NULL
GROUP BY p.id, t.id
ON DUPLICATE KEY UPDATE
    updated_at = NOW(),
    passing_host_count = VALUES(passing_host_count),
    failing_host_count = VALUES(failing_host_count);
```

This statement calculated passing/failing results and inserted them into a `policy_stats` summary table. Unfortunately, this query took over 30 seconds to execute. During this time, it locked the important `policy_membership` table, preventing other threads from writing to it.

## Reproducing slow SQL queries

Since we saw the issue in production, we needed to reproduce it in a test environment. We created a similar schema and loaded it with data. We used a Go script to populate the tables with dummy data: https://github.com/getvictor/mysql/blob/main/insert-with-subqueries-perf/main.go.

Initially, we used ten policies and ten teams with 10,000 hosts each, resulting in 100 inserted rows with the above query. However, the performance was only three to six seconds. Then, we increased the number of policies to 50, resulting in 500 inserted rows. The performance dropped to 30 to 60 seconds.

The above data made it clear that this query needed to be more scalable. As the `GROUP BY p.id, t.id` clause demonstrates, performance exponentially degrades with the number of policies and teams.

## Debugging slow SQL queries

MySQL has powerful tools called [EXPLAIN](https://dev.mysql.com/doc/refman/8.0/en/explain.html) and `EXPLAIN ANALYSE`. These tools show how MySQL executes a query and help identify performance bottlenecks. We ran `EXPLAIN ANALYSE` on the problematic query and viewed the results as a tree and a diagram.

{{< figure src="mysql-explain-tree.png" alt="MySQL EXPLAIN result in TREE format" title="MySQL EXPLAIN result in TREE format" >}}
\
{{< figure src="mysql-explain-diagram.png" alt="MySQL EXPLAIN result as a diagram" title="MySQL EXPLAIN result as a diagram" >}}

Although the `EXPLAIN` output was complex, it was clear that the `SELECT` subqueries were executing too many times.

## Fixing INSERT with subqueries performance

The first step was to separate the `INSERT` from the `SELECT`. The top `SELECT` subquery took most of the time. But, more importantly, the `SELECT` does not block other threads from updating the `policy_membership` table.

However, the single standalone `SELECT` subquery was still slow. In addition, memory usage could be high for many teams and policies.

We decided to process one policy row at a time. This reduced the time to complete an individual `SELECT` query to less than two seconds and limited the memory usage. We did not use a transaction to minimize locks. Not utilizing a transaction meant that the `INSERT` could fail if a parallel process deleted the policy. Also, the `INSERT` could overwrite a clearing of the `policy_stats` row. These drawbacks were acceptable, as they were rare cases.

```sql
SELECT
    p.id as policy_id,
    t.id AS inherited_team_id,
    (
        SELECT COUNT(*) 
        FROM policy_membership pm 
        INNER JOIN hosts h ON pm.host_id = h.id 
        WHERE pm.policy_id = p.id AND pm.passes = true AND h.team_id = t.id
    ) AS passing_host_count,
    (
        SELECT COUNT(*) 
        FROM policy_membership pm 
        INNER JOIN hosts h ON pm.host_id = h.id 
        WHERE pm.policy_id = p.id AND pm.passes = false AND h.team_id = t.id
    ) AS failing_host_count
FROM policies p
CROSS JOIN teams t
WHERE p.team_id IS NULL AND p.id = ?
GROUP BY t.id, p.id;
```

After each `SELECT`, we inserted the results into the `policy_stats` table.

```sql
INSERT INTO policy_stats (policy_id, inherited_team_id, passing_host_count, failing_host_count)
    VALUES (?, ?, ?, ?), ...
ON DUPLICATE KEY UPDATE
    updated_at = NOW(),
    passing_host_count = VALUES(passing_host_count),
    failing_host_count = VALUES(failing_host_count);
```

## Further reading about MySQL

- [MySQL deadlock on UPDATE/INSERT upsert pattern](../mysql-upsert-deadlock/)
- [Scaling DB performance using master slave replication](../mysql-master-slave-replication/)
- [Fully supporting Unicode and emojis in your app](../unicode-and-emoji-gotchas/)
- [SQL prepared statements are broken when scaling applications](../sql-prepared-statements-are-broken-when-scaling-applications/)

## MySQL code to populate DB on GitHub

The code to populate our test DB is available on GitHub at: https://github.com/getvictor/mysql/tree/main/insert-with-subqueries-perf

## MySQL query performance: INSERT with subqueries video

{{< youtube 9vulV3W-bp8 >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.
