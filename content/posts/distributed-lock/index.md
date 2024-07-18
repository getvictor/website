+++
title = 'Using a distributed lock in production distributed systems'
description = "Real-world example of distributed lock solving data inconsistency"
authors = ["Victor Lyuboslavsky"]
image = "distributed-lock-headline.png"
date = 2024-07-18
categories = ["Software Development"]
tags = ["Distributed Systems", "Locking", "Redis"]
draft = false
+++

This article will present a problem we encountered in our production distributed system and how we solved it using a
distributed lock.

## The problem -- data inconsistency

Recently, we started using the Google Calendar API to monitor calendar changes. However, we noticed that it is possible
to receive a second callback while processing the first one. This second callback can lead to data inconsistency, race
conditions, and deadlocks.

{{< figure src="data-consistency-problem.svg" title="Data inconsistency"
alt="Sequence diagram demonstrating data inconsistency issue without a distributed lock" >}}

In the above diagram, two servers, A and B, are processing calendar events. Server B receives a callback from Google
Calendar API stating that something has changed in the calendar. Google does not provide information about what event
changed, so server B must fetch the event of interest from the calendar. While server B is fetching the event, server A
also receives a callback. Server A also fetches the event from the calendar. Both servers now have the same event but
are unaware of each other's actions. Server B updates the event with new information. Server A also updates the event
with different details, potentially overwriting or duplicating Server B's changes. The calendar event is now in an
inconsistent state, as is the data in our database.

## What is a distributed lock?

A distributed lock is a mechanism that allows multiple servers to coordinate access to a shared resource. This mechanism
is widely used across the software industry to ensure data consistency in distributed systems.

In our case, we need to make sure that only one server is processing a calendar event at a time. The distributed lock
will prevent the second server from processing the event until the first server completes.

## Implementation of distributed lock

We implemented a distributed lock using Redis. Redis is an in-memory data structure store that can be used as a
database, cache, and message broker. To acquire the lock, our server sets a key in Redis with a unique value using the
[Redis SET command](https://redis.io/docs/latest/commands/set/).

```
SET mykey "myvalue" NX PX 60000
```

The `NX` option only sets the key if it does not exist. The `PX 60000` option sets the key's expiration time to 60
seconds. This ensures that the lock is released if the server crashes or does not release it in a timely manner.

To release the lock, we `EVAL` a Lua script that checks if the key's value matches the unique value set by the server.
If the values match, the script deletes the key using the
[Redis DEL command](https://redis.io/docs/latest/commands/del/).

```
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

We only release the lock if the value matches, ensuring that the server that acquired the lock releases it.

## Distributed lock solution

With the distributed lock in place, we can make sure that only one server is processing a calendar event at a time.

{{< figure src="distributed-lock-solution.svg" title="Using distributed lock with a processing queue"
alt="Sequence diagram demonstrating using distributed lock and a processing queue to solve data consistency issue" >}}

In the above diagram, server B receives a calendar callback and acquires a lock from Redis. Server A also gets a
callback but cannot acquire the lock since it has already been taken by Server B. Instead of waiting, Server A puts the
event in a processing queue. Once Server B finishes processing the event, it releases the lock. Server B then checks the
queue. Finding an event in the queue, the server starts a new worker process to process the events. The worker processes
all outstanding events in the queue and exits on completion.

## Waiting to acquire the lock

One issue we encountered was that another system process needed to acquire the lock. The process could keep trying to
obtain the lock, but there was no guarantee that it would be successful in a reasonable amount of time because it could
compete with other servers.

## Fairness in acquiring the lock

We implemented a fairness mechanism to ensure a priority process could acquire the lock.

{{< figure src="distributed-fair-lock.svg" title="Cron job is guaranteed to acquire the lock"
alt="Sequence diagram demonstrating using a fairness mechanism to acquire a distributed lock" >}}

In the above diagram, the worker process has acquired the lock. Another process, a cron job, also needs to acquire the
lock. The cron job is a priority process that needs to run at a specific time. The cron job tries to acquire the lock
but fails because the worker process has it. The cron job sets a key in Redis that indicates that it wants to acquire
the lock next. This action tells the other servers not to acquire the lock. The cron job then retries acquiring the lock
until it is successful.

## Distributed lock code on GitHub

We implemented the distributed lock logic in Go. The crucial part of the code is in the
[redis_lock.go](https://github.com/fleetdm/fleet/blob/7ae1fe95272fbbda7efe1e320552539768498839/server/service/redis_lock/redis_lock.go)
file.

## Distributed lock video

{{< youtube 833TqEPfF18 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
