+++
title = 'How to secure MySQL Docker container for Zero Trust'
description = "Increase the security of your MySQL Docker containers when running in a Zero Trust dev environment"
authors = ["Victor Lyuboslavsky"]
image = "mysql-docker-headline.png"
date = 2024-08-20
categories = ["DevOps & Infrastructure", "Database Administration"]
tags = ["MySQL", "Docker"]
draft = false
+++

- [Securing database secrets with Docker secrets](#securing-database-secrets-with-docker-secrets)
- [Securing database secrets with SQL commands](#securing-database-secrets-with-sql-commands)

## What is a Zero Trust development environment?

[Zero Trust](https://www.cloudflare.com/learning/security/glossary/what-is-zero-trust-security/) is a security model
that assumes no trust, even inside the network. Every request is authenticated, authorized, and encrypted in a Zero
Trust environment. This approach helps protect against data breaches and insider threats.

In our example use case, we create a development environment in a cloud instance, which includes a MySQL database
running in a Docker container. We need to be able to access the MySQL database from our local machine for development
purposes. However, the database may contain sensitive data, such as API keys or user passwords. We want to secure the
MySQL database to prevent unauthorized access.

We want to make sure that the MySQL database is not easily accessible from the internet. In addition, we want to limit
the exposure of database credentials.

## Launching MySQL Docker container

We can run a MySQL database in a Docker container using the
[official MySQL Docker image](https://hub.docker.com/_/mysql). We create `docker-compose.yml` like:

```yaml
services:
  mysql:
    image: mysql:8.0
    command: [
        "mysqld",
        "--datadir=/tmp/mysqldata",
      ]
    environment:
      MYSQL_ROOT_PASSWORD: toor
      MYSQL_DATABASE: fleet
      MYSQL_USER: fleet
      MYSQL_PASSWORD: insecure
    ports:
      - "3306:3306"
```

And we run `docker-compose up` to start the MySQL database.

We can access the MySQL database by using the MySQL client:

```bash
mysql -h 127.0.0.1 -P 3306 -uroot -ptoor
```

As we can see, the passwords are stored in plain text in the `docker-compose.yml` file. We want to avoid storing
sensitive data in plain text.

## Securing database secrets with Docker secrets {#securing-database-secrets-with-docker-secrets}

[Docker secrets](https://docs.docker.com/compose/use-secrets/) allow us to store sensitive data, such as passwords,
securely. We can create secrets and use them in the `docker-compose.yml` file.

```yaml
secrets:
  mysql_root_password:
    file: ./mysql_root_password.txt
  mysql_password:
    environment: MYSQL_PASSWORD
services:
  mysql:
    image: mysql:8.0
    command: [
        "mysqld",
        "--datadir=/tmp/mysqldata",
      ]
    secrets:
      - mysql_root_password
      - mysql_password
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/mysql_root_password
      MYSQL_DATABASE: fleet
      MYSQL_USER: fleet
      MYSQL_PASSWORD_FILE: /run/secrets/mysql_password
    ports:
      - "3306:3306"
```

We create a `mysql_root_password.txt` file and run `MYSQL_PASSWORD=insecure docker-compose up` to start the MySQL
database.

The above example shows that the MySQL root password is stored in a file, and the MySQL password is passed as an
environment variable. Although this approach may be an improvement, it is not secure for a Zero Trust environment. A
user with access to the file system can read the secrets, and environment variables can be read by anyone who can run
the `ps` command, like: `ps eww <docker compose process ID>`.

In addition, a user can dump the secrets from the Docker container by running:

```
docker exec <container ID> cat /run/secrets/mysql_root_password
```

## Securing database secrets with SQL commands {#securing-database-secrets-with-sql-commands}

To secure the MySQL database without exposing the secrets on the server, we can use MySQL commands to set the passwords.
We spin up MySQL with the following `docker-compose.yml`:

```yaml
services:
  mysql:
    image: mysql:8.0
    command: [
        "mysqld",
        "--datadir=/tmp/mysqldata",
      ]
    environment:
      MYSQL_ROOT_PASSWORD: toor
      MYSQL_ONETIME_PASSWORD: true
    ports:
      - "3306:3306"
```

We set the root password and marked the root user as expired with `MYSQL_ONETIME_PASSWORD: true`.

Now, as the second step, we can run the following commands to set the passwords:

```
echo \
"ALTER USER root IDENTIFIED BY '$(op read op://employee/DEMO_SERVER/MYSQL_ROOT_PASSWORD)';" \
"CREATE DATABASE fleet;" \
"CREATE USER 'fleet'@'%' IDENTIFIED BY '$(op read op://employee/DEMO_SERVER/MYSQL_PASSWORD)';" \
"GRANT ALL PRIVILEGES ON fleet.* TO 'fleet'@'%';" \
"FLUSH PRIVILEGES;" \
| mysql -h 127.0.0.1 -P 3306 -uroot -ptoor --connect-expired-password
```

In the above command, we use [1Password](https://support.1password.com/command-line/) as our secrets manager. We read
the secrets from 1Password and pass them to the MySQL client to set the passwords.

## Additional security considerations

This article focused on securing the MySQL passwords. However, there are additional security considerations when running
MySQL in a Zero Trust environment:

- Encrypting sensitive data -- all sensitive data should be encrypted when stored in the database
- Limiting access to specific IPs -- we can add a server firewall to restrict access to the MySQL port

## Further reading

Recently, we wrote [how to use STDIN to read your program arguments](../get-args-from-stdin).

We've previously written about [MySQL master-slave replication](../mysql-master-slave-replication). You can use MySQL
replication to create a high-availability setup for your MySQL databases.

## Watch how to secure a MySQL Docker container for Zero Trust

{{< youtube GgEPIvFbnT0 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
