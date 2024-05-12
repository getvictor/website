+++
title = 'Get current telemetry from your devices with live queries'
description = "Query devices with Fleet"
image = "Live Query.png"
date = 2023-12-29
categories = ["Security"]
tags = ["Fleet", "Cyber Security", "Under the Hood"]
draft = false
+++

{{< youtube Jh14hNjW0Uo >}}

[Fleet](https://fleetdm.com/) is an open-source platform for managing and gathering telemetry from devices such as laptops, desktops, VMs, etc. [Osquery](https://www.osquery.io/) agents run on these devices and report to the Fleet server. One of Fleet’s features is the ability to query information from the devices in near real-time, called _live queries_. This article discusses how live queries work “under the hood.”


## Why a live query?

Live queries enable administrators to ask near real-time questions of all online devices, such as checking the encryption status of SSH keys across endpoints, or obtaining the uptime of each server within their purview. This enables them to promptly identify and address any issues, thereby reducing downtime and maintaining operational efficiency. These tasks, which would be time-consuming and complex if done manually, are streamlined through live queries, offering real-time insights into the status and posture of the entire fleet of devices helping IT and security.


## Live queries under the hood

Live queries can be run from the web UI, the command-line interface called `fleetctl`, or the REST API. The user creates a query and selects which devices will run that query. Here is an example using `fleetctl` to obtain the operating system name and version for all devices:


```shell
fleetctl query --query "select name, version from os_version;" --labels "All Hosts"
```


When a client initiates a live query, the server first creates a **Query Campaign** record in the MySQL database. A Fleet deployment consists of several servers behind a load balancer, so storing the record in the DB makes all servers aware of the new query campaign.

{{< figure src="Live Query.png" title="Query campaign" alt="Query campaign" >}}

As devices called **Hosts** in Fleet check in with the servers, they receive instructions to run a query. For example:


```json
{
    "queries": {
        "fleet_distributed_query_140": "SELECT name, version FROM os_version;"
    },
    "discovery": {
        "fleet_distributed_query_140": "SELECT 1"
    }
}
```


Then, the osquery agents run the actual query on their host, and write the result back to a Fleet server. As a server receives the result, it publishes it to the common cache using [Redis Pub/Sub](https://redis.io/docs/interact/pubsub/).

Only the one server communicating with the client subscribes to the results. It processes the data from the cache, keeps track of how many hosts reported back, and communicates results back to the client. The web UI and `fleetctl` interfaces use a [WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API), and results are reported as they come in. The REST API, on the other hand, only sends a response after all online hosts have reported their query results.


## Discover more

Fleet’s live query feature represents a powerful tool in the arsenal of IT and security administrators. By harnessing the capabilities of live queries, tasks that once required extensive manual effort can now be executed swiftly and efficiently. This real-time querying ability enhances operational efficiency and significantly bolsters security and compliance measures across a range of devices.

The integration of Fleet with Osquery agents, the flexibility offered by interfaces like the web UI, `fleetctl`, and the REST API, and the efficient data handling through mechanisms like Redis Pub/Sub and WebSockets API all come together to create a robust, real-time telemetry gathering system. This system is designed to keep you informed about the current state of your device fleet, helping you make informed decisions quickly.

As you reflect on the capabilities of live queries with Fleet, consider your network environment's unique challenges and needs. **What questions could live queries help you answer about your devices?** Whether it's security audits, performance monitoring, or compliance checks, live queries offer a dynamic solution to address these concerns.

We encourage you to explore the possibilities and share your thoughts or questions. Perhaps you’re facing a specific query challenge or an innovative use case you’ve discovered. Whatever it may be, the world of live queries is vast and ripe for exploration. Join us in [Fleet’s Slack forums](https://fleetdm.com/support) to engage with a community of like-minded professionals and deepen your understanding of what live queries can achieve in your environment.

API Documentation:

* [Run live query with REST API](https://fleetdm.com/docs/rest-api/rest-api#run-live-query)
* [Run live query with WebSockets](https://github.com/fleetdm/fleet/blob/6fd06d648601edd89c01e25426e2e35ff2a8a37b/docs/Contributing/API-for-contributors.md#run-live-query)

_This article originally appeared in [Fleet's blog](https://fleetdm.com/guides/get-current-telemetry-from-your-devices-with-live-queries)._
