+++
title = 'MySQL deadlock on UPDATE/INSERT upsert pattern'
description = "Using the UPDATE/INSERT upsert pattern can lead to MySQL deadlocks"
image = "MySQL deadlock.png"
date = 2024-04-28
categories = ["Database Administration"]
tags = ["SQL Developer", "MySQL", "Deadlock"]
draft = false
+++

## What is an SQL deadlock?

A deadlock occurs when two or more SQL transactions are waiting for each other to release locks. This can occur when two transactions have locks on separate resources and each is waiting for the other to release its lock.

## What is an upsert?

An upsert combines the words "update" and "insert." It is a database operation that inserts a new row into a table if the row does not exist or updates the row if it already exists.

## INSERT/UPDATE upsert pattern

One common way to implement an upsert operation in MySQL is to use the following pattern:

```sql
UPDATE table_name SET column1 = value1, column2 = value2 WHERE id = ?;
-- If the UPDATE statement does not affect any rows, insert a new row:
INSERT INTO table_name (id, column1, column2) VALUES (?, value1, value2);
```

[UPDATE](https://dev.mysql.com/doc/refman/8.0/en/update.html) returns the number of rows that were actually changed.

This UPDATE/INSERT pattern is optimized for frequent updates and rare inserts. However, it can lead to deadlocks when multiple transactions try to insert simultaneously.

## MySQL deadlock example

We assume the default transaction isolation level of [REPEATABLE READ](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_repeatable-read). Given the following table with one row:

```sql
CREATE TABLE my_table (
    id int(10) unsigned NOT NULL,
    amount int(10) unsigned NOT NULL,
    PRIMARY KEY (id)
);
INSERT INTO my_table (id, amount) VALUES (1, 1);
```

One transaction executes the following SQL:

```sql
UPDATE my_table SET amount = 2 WHERE id = 2;
```

Another transaction executes the following SQL:

```sql
UPDATE my_table SET amount = 3 WHERE id = 3;
INSERT INTO my_table (id, amount) VALUES (3, 3);
```

At this point, the second transaction is waiting for the first transaction to release the lock.

The first transaction then executes the following SQL:

```sql
INSERT INTO my_table (id, amount) VALUES (2, 2);
```

Causing a deadlock:

```
[40001][1213] Deadlock found when trying to get lock; try restarting transaction
```

### Why does the deadlock occur?

The deadlock occurs because both transactions set [next-key locks](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-next-key-locks) on the rows they attempted to update. Since the rows they attempted to update did not exist, the lock is set on the "supremum" pseudo-record. This pseudo-record has a value higher than any value actually in the index. This lock prevents the other transaction from inserting the row it needs.

## Debugging MySQL deadlocks

To view the last deadlock detected by MySQL, you can use:

```sql
SHOW ENGINE INNODB STATUS;
```

The output will contain a section like this:

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-04-28 12:29:17 281472351068032
*** (1) TRANSACTION:
TRANSACTION 1638819, ACTIVE 7 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1128, 2 row lock(s)
MySQL thread id 97926, OS thread handle 281471580295040, query id 24192112 192.168.65.1 root update
/* ApplicationName=DataGrip 2024.1 */ INSERT INTO my_table (id, amount) VALUES (3, 3)

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 158 page no 4 n bits 72 index PRIMARY of table `test`.`my_table` trx id 1638819 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 158 page no 4 n bits 72 index PRIMARY of table `test`.`my_table` trx id 1638819 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


*** (2) TRANSACTION:
TRANSACTION 1638812, ACTIVE 13 sec inserting
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1128, 2 row lock(s)
MySQL thread id 97875, OS thread handle 281471585578880, query id 24192285 192.168.65.1 root update
/* ApplicationName=DataGrip 2024.1 */ INSERT INTO my_table (id, amount) VALUES (2, 2)

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 158 page no 4 n bits 72 index PRIMARY of table `test`.`my_table` trx id 1638812 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 158 page no 4 n bits 72 index PRIMARY of table `test`.`my_table` trx id 1638812 lock_mode X insert intention waiting
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;
```

We can see the supremum locks held by both transactions: ` 0: len 8; hex 73757072656d756d; asc supremum;;`.

Another way to debug MySQL deadlocks is to enable the [innodb_print_all_deadlocks](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_print_all_deadlocks) option. This option prints all deadlocks to the error log.

## Preventing the UPDATE/INSERT deadlock

One way to prevent the deadlock is to use the [INSERT ... ON DUPLICATE KEY UPDATE](https://dev.mysql.com/doc/refman/8.0/en/insert-on-duplicate.html) pattern. This syntax allows you to insert a new row or update an existing row in a single statement. However, it is slower than an UPDATE and always increments the auto-increment value if present.

Another way is to roll back the transaction once we know that the UPDATE did not affect any rows. This avoids the deadlock by not holding the lock while we insert the new row. After the rollback, we can retry the transaction using the above `INSERT ... ON DUPLICATE KEY UPDATE` pattern.

A third way is not to use transactions. In this case, the locks are released immediately after the statement is executed. However, this approach may not be suitable for all use cases.

## Conclusion

The UPDATE/INSERT upsert pattern can lead to MySQL deadlocks when multiple transactions try to insert simultaneously. To prevent deadlocks, consider using the `INSERT ... ON DUPLICATE KEY UPDATE` pattern, rolling back the transaction, or not using transactions.

## MySQL deadlock on UPDATE/INSERT upsert pattern video

{{< youtube ADerRg7tzag >}}

## Other articles related to MySQL

- [Optimize MySQL query performance: INSERT with subqueries](../mysql-query-performance-insert-subqueries/)
- [Fully supporting Unicode and emojis in your app](../unicode-and-emoji-gotchas/)
- [SQL prepared statements are broken when scaling applications](../sql-prepared-statements-are-broken-when-scaling-applications/)

*Note:* If you want to comment on this article, please do so on the YouTube video.

