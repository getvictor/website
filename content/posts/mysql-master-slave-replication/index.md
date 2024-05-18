+++
title = 'Create a MySQL slave replica in 4 short steps'
description = "Scale your database using master-slave replication"
image = "mysql-master-slave-replication.png"
date = 2024-05-18
categories = ["Database Administration"]
tags = ["MySQL", "Database Scaling"]
draft = false
+++

## Introduction

In this article, we will create a MySQL slave replica. A MySQL slave is a read-only copy of the master database. Using MySQL replication, the slave database is kept in sync with the master database.

The steps we will follow are:
1. [Spin up MySQL master and slave databases](#create-mysql-master-and-slave-databases)
2. [Create a user for replication](#create-db-user-for-replication)
3. [Obtain master binary log coordinates](#retrieve-master-binary-log-coordinates)
4. [Configure slave and start replication](#configure-slave-and-start-replication)

## What is database replication?

Database replication is a process that allows data from one database server (the master) to be copied to one or more database servers (the slaves or replicas). Replication is asynchronous, meaning that the slave does not need to be connected to the master constantly. The replica can catch up with the master when it is available.

Database replicas are used for:
- Scaling read operations
- High availability
- Disaster recovery

MySQL implements replication using the binary log. The master server writes changes to the binary log, and the slave server reads the binary log and applies the changes to its database.

## Create MySQL master and slave databases {#create-mysql-master-and-slave-databases}

We will use Docker to create the MySQL master and slave databases. We will use the [official MySQL Docker image](https://hub.docker.com/_/mysql). The master database will run on port 3308, and the slave database will run on port 3309.

We run `docker compose up` using the following `docker-compose.yml` file:

{{< gist getvictor 92ce5a8541ce27a1ea36f9eb7feb0344 >}}

## Create a DB user for replication {#create-db-user-for-replication}

Replication in MySQL requires a user with the `REPLICATION SLAVE` privilege. We will create a user named `replicator` with the password `rotacilper`.

Connect to the master database using the MySQL client:

```bash
mysql --host 127.0.0.1 --port 3308 -uroot -ptoor
```

Create the `replicator` user and grant the `REPLICATION SLAVE` privilege:

```sql
CREATE USER 'replicator'@'%' IDENTIFIED BY 'rotacilper';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
FLUSH PRIVILEGES;
```

## Retrieve master binary log coordinates {#retrieve-master-binary-log-coordinates}

For the slave to start replication, it needs to know the master's binary log file and position. We can obtain this information using the MySQL client which we opened in the previous step.

```sql
SHOW MASTER STATUS;
```

The output will look like this:

```
+------------+----------+--------------+------------------+-------------------+
| File       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------+----------+--------------+------------------+-------------------+
| bin.000003 |      861 |              |                  |                   |
+------------+----------+--------------+------------------+-------------------+
1 row in set (0.01 sec)
```

We must remember the `File` and `Position` values for the next step.

## Configure slave and start replication {#configure-slave-and-start-replication}

Now, we will connect to the slave database and configure it to replicate from the master database.

```bash
mysql --host 127.0.0.1 --port 3309 -uroot -ptoor
```

Use the `CHANGE MASTER TO` command to configure the slave to replicate from the master. Replace `MASTER_LOG_FILE` and `MASTER_LOG_POS` with the values obtained in the previous step.

```sql
CHANGE MASTER TO
  MASTER_HOST='mysql_master',
  MASTER_PORT=3306,
  MASTER_USER='replicator',
  MASTER_PASSWORD='rotacilper',
  MASTER_LOG_FILE='bin.000003',
  MASTER_LOG_POS=861,
  GET_MASTER_PUBLIC_KEY=1;
```

`MASTER_HOST` is the hostname of the master, which matches the docker service name. The `GET_MASTER_PUBLIC_KEY` option is needed for MySQL 8.0 `caching_sha2_password` authentication.

Finally, start the slave:

```sql
START SLAVE;
```

The slave will now start replicating data from the master database. You can check the replication status using the `SHOW REPLICA STATUS\G` command.

We can create a table with data on the master database and check if it is replicated to the slave database:

```sql
USE test;
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(255));
INSERT INTO users VALUES (1, 'Alice');
```

## Further reading on database scaling

- Recently, we wrote about [optimizing a MySQL INSERT with subqueries](../mysql-query-performance-insert-subqueries).
- In the past, we encountered a [memory issue with MySQL prepared statements when scaling applications](../sql-prepared-statements-are-broken-when-scaling-applications).

## Follow along with the MySQL master-slave replication on video

{{< youtube nMbb1199HQU >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.
