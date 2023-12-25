+++
title = 'SQL prepared statements are broken when scaling applications'
description = 'We hit a snag with SQL prepared statements recently'
image = "cover.png"
date = 2023-12-14
tags = ["sqldeveloper", "mysql", "backenddevelopers"]
draft = false
+++

{{< youtube JHoEKmNj8t8 >}}

A prepared statement is a feature of modern databases intended to help execute the same SQL
statement multiple times. For example, the following statement is a prepared statement:

```sql
SELECT id, name FROM users WHERE email = ?;
```

The presence of an unspecified parameter, labeled “?”, makes it a prepared statement. When a
prepared statement is sent to the database, it is compiled, optimized, and stored in memory on the
database server. Subsequently, the client application may execute the same prepared statement
multiple times with different parameter values. This results in a speedup.

Prepared statements are well suited for long and complex queries that require significant
compilation and optimization times. They are kept prepared on the DB server, and the application
must only pass the parameters to execute them.

Another benefit of using prepared statements is the protection they provide
against [SQL injection](https://owasp.org/www-community/attacks/SQL_Injection). The application does
not need to properly escape the parameter values provided to the statement. Because of this
protection, many experts recommend always using prepared statements for accessing the database.

However, by always using prepared statements for accessing the database, we force the SQL driver to
send the extra prepare command for every ad-hoc statement we execute. The driver sends the following
commands:

1. Prepare the statement
2. Execute statement with given parameters
3. Close the statement (and deallocate the prepared statement created above)

Another issue with prepared statements is the memory requirement. In large application deployments
with large numbers of connections, prepared statements can crash your environment. This issue
happened to one of our customers.

A prepared statement is only valid for a single session, which typically maps to a single database
connection. If the application runs multiple servers, with many connections, it may end up storing a
prepared statement for each one of those sessions.

For example, given 100 servers with 100 connections each, we have 10,000 connections to the
database. Assuming a memory requirement of 50 KB per prepared statement (derived from the
following [article](https://blog.searce.com/how-max-prepared-stmt-count-bring-down-the-production-mysql-system-6ca28e577663)),
we arrive at the maximum memory requirement of:

```
10,000 * 50 KB = 500 MB per single saved prepared statement
```

Some databases also have limits on the number of prepared statements. MySQL’s
[max_prepared_stmt_count](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_max_prepared_stmt_count) defaults to 16,382 for the entire server. Yes, this is a global limit, and
**not** per session. In the above example, if the application uses prepared statements for every
database access, then each database connection will always be using up 1 short-lived prepared
statement. A short-lived prepared statement is the prepared statement, as we described above, that
will be created for the purposes of executing one statement, and then immediately deallocated
afterwards. This means the above application running with a default MySQL config **cannot explicitly
save any prepared statements** -- 10,000 transient prepared statements + 10,000 saved prepared
statements is greater than the max_prepared_stmt_count of 16,382.

This is **extremely inconvenient** for application developers, because they must keep track of:
- The number of saved prepared statements they are using
- How many application servers are running
- How many database connections each server has
- The prepared statement limits of the database 

This detail can easily be overlooked when scaling applications.

In the end, is it really worth using prepared statements, and especially saved prepared statements, in your application? Yes, saved prepared statements can offer performance advantages, especially for complex queries executed frequently. However they must also be kept in check.

A few ways to mitigate prepared statement issues for large application deployments include:
- Limit the number of database connections per application server
- Increase the prepared statement limit on the database server(s)
- Limit the maximum lifespan of connections. When closing a connection, the database will deallocate all prepared statements on that connection.
