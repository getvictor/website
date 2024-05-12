+++
title = 'Physical security meets cybersecurity with Matter'
description = 'Will Matter change the landscape of physical security?'
image = "cover.png"
date = 2023-12-20
categories = ["Security"]
tags = ["Physical Security", "Cyber Security", "Matter"]
draft = false
+++

{{< youtube mIVsLTrUork >}}

Matter is a recent open-source standard for connecting devices such as light switches, door locks,
motion sensors, and many others. The major goals of the standard are compatibility and
interoperability. This means that you will no longer need to be an expert hacker when trying to
control devices from multiple manufacturers under a single application. Apple, Amazon, and Google
are some of the major members driving the standard. This is great news for the majority of adopters
who haven’t yet fully embraced home automation and security.

{{< figure src="matter.jpeg" >}}

The Matter specification is published by
the [Connectivity Standards Alliance](https://csa-iot.org/) (CSA) and includes a
[software development kit](https://github.com/project-chip/connectedhomeip). Version 1.0 of the
specification was released in October of 2022. In 2023,
we saw a slew of new devices and software upgrades compatible with Matter. Version 1.2 of the
specification was published in October of 2023. However, this latest specification is still missing
support for a few important device categories such as cameras and major appliances. Cameras are a
top priority for the CSA, and we may see Matter-compatible cameras in 2024.

Matter is an important step for the management of IoT devices because it finally brings true
interoperability where it has been sorely missing for so many years. No longer will device
manufacturers need to decide and budget precious software resources to support Amazon Alexa, Google
Home, Apple HomeKit, or another connectivity hub. Customers will no longer be locked into using one
of the major home automation providers. And home automation solutions from smaller companies will
come onto the market.

An important feature of Matter is **multi-admin**, which means that devices can be read and
controlled by multiple clients. In Matter terminology, the device, such as a motion sensor, is
called a server or node, and the applications controlling it are called clients. For example, a
light switch may be simultaneously controlled by the manufacturer’s app, by Alexa, and by the user's
hand-written custom API client.

Multi-admin support means that a home or business may use one application to control their locks,
switches, and security sensors, and another application for reading telemetry from those same
devices. Businesses will find it easier to integrate physical security with cyber security. For
example, suppose a business’s device management server uses Matter to subscribe to the office door
lock. It receives an alert that _User A_ has entered their code. Afterwards, via regular scheduled
telemetry, it notices a successful login to _Computer B_. The business SIEM (security information and
event management) system should immediately flag this suspicious sequence of events.

{{< figure src="cover.png" alt="Physical security with Matter">}}

Of course, the example above can be accomplished today by writing some custom code or using a third party integration. What Matter brings is scalability to such security approaches. The code and integration will no longer need to be redone for each new device and version that comes onto the market.
