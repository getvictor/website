+++
title = 'Building a webhook flow with Tines'
description = "Trigger an action from a webhook callback with Tines"
image = "tines-fleet-webhook-workflow.png"
date = 2024-03-29
tags = ["webhook", "Tines", "MDM", "Fleet"]
draft = true
+++

## What is a webhook?

A webhook is a way for one application to send data to another application in real time. It is a simple way to trigger an action based on an event. In other words, a webhook is a custom HTTP callback.

## What is Tines?

[Tines](https://www.tines.io/) is a no-code automation platform that allows you to automate repetitive tasks. It is a powerful tool that can be used to automate workflows, such as sending emails, creating tickets, and updating databases.

## What is Fleet?

[Fleet](https://fleetdm.com/) is an open-source platform for managing and gathering telemetry from devices such as laptops, desktops, VMs, etc. [Osquery](https://www.osquery.io/) agents run on these devices and report to the Fleet server.

## Our example IT workflow

In this article, we will build a webhook flow with Tines. When a device has an outdated OS version, Tines will receive a webhook callback from Fleet. Tines will then send an MDM (Mobile Device Management) command to the device to update the device's OS version.

Fleet will send a callback via its calendar integration feature. Fleet can put a "Downtime" event on the device user's calendar. This event warns the device owner that their computer will be restarted to remediate one or more failing policies. During the calendar event time, Fleet sends a webhook. The IT admin must set up a flow to remediate the failing policy. This article is an example of one such flow.

## Getting started -- webhook action

First, we create a new Tines story. A story is a sequence of actions that are executed in order. Next, we add a webhook action to the story. The webhook action listens for incoming webhooks. The webhook will contain a JSON body.

{{< figure src="1-tines-webhook.png" title="Tines webhook action" alt="Tines webhook action" >}}

## Handling errors

Often, webhooks may contain error messages if there is an issue with the configuration, flow, etc. In this example, we add a trigger action that checks whether the webhook body contains an error. Specifically, our action checks whether the webhook body contains a non-empty "error" field.

{{< figure src="2-tines-error-handling.png" title="Tines trigger action checking for an error" alt="Tines trigger action checking for an error" >}}

We leave this error-handling portion of the story as a stub. In the future, we can expand it by sending an email or triggering other actions.

## Checking whether webhook indicates an outdated OS

At the same time, we also check whether the webhook was triggered by a policy indicating an outdated OS. From previous testing, we know that the webhook payload will look like this:

```json
{
  "timestamp": "2024-03-28T13:57:31.668954-05:00",
  "host_id": 11058,
  "host_display_name": "Victor's Virtual Machine (2)",
  "host_serial_number": "Z5C4L7GKY0",
  "failing_policies": [
    {
      "id": 479,
      "name": "macOS - OS version up to date"
    }
  ]
}
```

The payload contains:
  - The device's ID (host ID).
  - Display name.
  - Serial number.
  - A list of failing policies.

We are interested in the failing policies. When one of the failing policies contains a policy named "macOS - OS version up to date," we know that the device's OS is outdated. Hence, we create a trigger that looks for this policy.

{{< figure src="3-tines-os-version-trigger.png" title="Tines trigger action checking for an outdated OS" alt="Tines trigger action checking for an outdated OS" >}}

We use the following formula, which loops over all policies and will only allow the workflow to proceed if true:

```
IF(FIND(calendar_webhook.body.failing_policies, LAMBDA(item, item.name = "macOS - OS version up to date")).id > 0, TRUE)
```

## Getting device details from Fleet

Next, we need to get more details about the device from Fleet. Devices are called hosts in Fleet. We add an "HTTP Request" action to the story. The action makes a GET request to the Fleet API to get the device details. We use the host ID from the webhook payload. We are looking for the device's UUID, which we need to send the OS update MDM command.

{{< figure src="4-tines-get-host-request.png" title="Tines HTTP Request action to get Fleet device details" alt="Tines HTTP Request action to get Fleet device details" >}}

To access Fleet's API, we need to provide an API key. We store the API key as a CREDENTIAL in the current story. The API key should belong to an API-only user in Fleet so that the key does not reset when the user logs out.

{{< figure src="5-tines-credential.png" title="Add credential to Tines story" alt="Add credential to Tines story" >}}

## Creating MDM command payload to update OS version

We can create the MDM payload now that we have the device's UUID. The payload contains the command to update the OS version. We use the [ScheduleOSUpdate](https://developer.apple.com/documentation/devicemanagement/schedule_an_os_update?language=objc) command from Apple's MDM protocol.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Command</key>
    <dict>
        <key>RequestType</key>
        <string>ScheduleOSUpdate</string>
        <key>Updates</key>
        <array>
            <dict>
                <key>InstallAction</key>
                <string>InstallASAP</string>
                <key>ProductVersion</key>
                <string>14.4.1</string>
            </dict>
        </array>
    </dict>
    <key>CommandUUID</key>
    <string><<UUID()>></string>
</dict>
</plist>
```

This command will download macOS 14.4.1, install it, and pop up a 60-second countdown dialog box before restarting the device. Note that the `<<UUID()>>` Tines function creates a unique UUID for this MDM command.

{{< figure src="6-tines-create-mdm-command.png" title="Tines event to create ScheduleOSUpdate MDM command" alt="Tines event to create ScheduleOSUpdate MDM command" >}}

The Fleet API requires the command to be sent as a base64-encoded string. We add a "Base64 Encode" action to the story to encode the XML payload. It uses the Tines `BASE64_ENCODE` function.

{{< figure src="7-tines-base64-encode.png" title="Tines Base64 Encode event" alt="Tines Base64 Encode event" >}}

## Run MDM command on device

Finally, we send the MDM command to the device. We add another "HTTP Request" action to the story. The action makes a POST request to the Fleet API to send the MDM command to the device.

{{< figure src="8-tines-run-mdm-command.png" title="Tines HTTP Request action to run MDM command on device" alt="Tines HTTP Request action to run MDM command on device" >}}

The MDM command will run on the device, downloading and installing the OS update.

{{< figure src="9-macos-device-restart.png" title="macOS restart notification after OS update" alt="macOS restart notification after OS update" >}}

## Conclusion

In this article, we built a webhook flow with Tines. We received a webhook callback from Fleet when a device had an outdated OS version. We then sent an MDM command to the device to update the OS version. This example demonstrates how Tines can automate workflows and tasks in IT environments.

## Building a webhook flow with Tines video

{{< youtube TODO >}}

*Note:* If you want to comment on this article, please do so on the YouTube video.
