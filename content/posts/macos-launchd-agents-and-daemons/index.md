+++
title = "What are launchd agents and daemons on macOS?"
description = "Understand macOS launchd, agents, daemons, and property list files. Discover how to locate the plist file for a running process."
authors = ["Victor Lyuboslavsky"]
image = "macos-agents-and-daemons.png"
date = 2024-12-11
categories = ["DevOps & Infrastructure"]
tags = ["macOS", "Agents"]
draft = false
+++

- [How to find the plist file for a running process](#how-to-find-the-plist-file-for-a-running-process)
- [Create and edit .plist files with PlistBuddy](#create-and-edit-plist-files-with-plistbuddy)

## What is launchd?

`launchd` is a macOS system service manager that starts, stops, and manages daemons, agents, and other processes. It is
the first process the kernel starts and is responsible for starting all other processes on the system.

If you go to `Activity Monitor` on your Mac and `View` > `All Processes, Hierarchically`, you will see that all
processes are children of `launchd`.

{{< figure src="activity-monitor.png" alt="The top process name is kernel_task, its child is launchd, and everything else is a child of launchd.">}}

## What are launchd agents and daemons?

`launchd` can start and manage agents and daemons.

### Daemons

Daemons are background processes that run without a user interface. They typically start at boot time and run
continuously in the background. One example of a daemon is Apple's `timed` time synchronization daemon, which maintains
system clock accuracy by synchronizing the clock with reference clocks over the network. Another example is a device
management daemon, such as [Fleet's `orbit`](https://fleetdm.com/docs/get-started/anatomy#orbit), which manages the
device's configuration and security settings.

### Agents

Agents are similar to daemons but run in the context of a user session. They are started when a user logs in and can
interact with the user interface. Agents are helpful for tasks that need to run in the background but also need to
communicate with the user. For example, a security agent can check the system's state and notify the user if they fail a
corporate security policy.

Agents may or may not have a user interface. Many 3rd party agents run in the background and provide a menu bar icon to
configure the agent's behavior.

## How are agents and daemons configured with plist?

`launchd` uses property list (`.plist`) files to define the configuration of agents and daemons. These files specify the
program to run, the arguments to pass, the environment variables to set, and other settings.

Here is an example of a `.plist` file for a launchd daemon:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>EnvironmentVariables</key>
    <dict>
       <key>ORBIT_ENROLL_SECRET_PATH</key>
       <string>/opt/orbit/secret.txt</string>
       <key>ORBIT_FLEET_URL</key>
       <string>https://dogfood.fleetdm.com</string>
       <key>ORBIT_ENABLE_SCRIPTS</key>
       <string>true</string>
       <key>ORBIT_ORBIT_CHANNEL</key>
       <string>stable</string>
       <key>ORBIT_OSQUERYD_CHANNEL</key>
       <string>stable</string>
       <key>ORBIT_UPDATE_URL</key>
       <string>https://updates.fleetdm.com</string>
       <key>ORBIT_FLEET_DESKTOP</key>
       <string>true</string>
       <key>ORBIT_DESKTOP_CHANNEL</key>
       <string>stable</string>
       <key>ORBIT_UPDATE_INTERVAL</key>
       <string>15m0s</string>
    </dict>
    <key>KeepAlive</key>
    <true/>
    <key>Label</key>
    <string>com.fleetdm.orbit</string>
    <key>ProgramArguments</key>
    <array>
       <string>/opt/orbit/bin/orbit/orbit</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardErrorPath</key>
    <string>/var/log/orbit/orbit.stderr.log</string>
    <key>StandardOutPath</key>
    <string>/var/log/orbit/orbit.stdout.log</string>
    <key>ThrottleInterval</key>
    <integer>10</integer>
</dict>
</plist>
```

The typical locations for agent and daemon `.plist` files are:

| Type           | Location                        |
| -------------- | ------------------------------- |
| User Agents    | `~/Library/LaunchAgents`        |
| Global Agents  | `/Library/LaunchAgents`         |
| System Agents  | `/System/Library/LaunchAgents`  |
| Global Daemons | `/Library/LaunchDaemons`        |
| System Daemons | `/System/Library/LaunchDaemons` |

_Note:_ In rare cases, the `.plist` files may be located in other directories or missing entirely.

### How to view the contents of a `.plist` file

`.plist` files come in several formats, including binary, XML, and JSON.

You can view the contents of a `.plist` file using the `plutil` (property list utility) command. For example:

```shell
plutil -p /System/Library/LaunchDaemons/com.apple.analyticsd.plist
```

`plutil` can also convert between different `.plist` formats. For example, to convert a binary `.plist` file to XML,
run:

```shell
cp /System/Library/LaunchDaemons/com.apple.analyticsd.plist my.plist
plutil -convert xml1 my.plist
```

### Can I use a `.plist` file for cron-like scheduling?

Yes, you can use `launchd` to schedule tasks in a `.plist` file. `launchd` is the recommended alternative to `cron` on
macOS. The `StartCalendarInterval` key specifies when the task should run. For example, to run a task every day at 5 AM,
you can add the following to your `.plist` file:

```xml
<key>StartCalendarInterval</key>
<dict>
    <key>Hour</key>
    <integer>5</integer>
    <key>Minute</key>
    <integer>0</integer>
</dict>
```

## How to find the plist file for a running process

Suppose you identified a process running on your Mac from `Activity Monitor` and want to find the `.plist` file that
started it. The process should be a child of `launchd`.

To find the identifier of a running process, you can use the `launchctl` command. The `launchctl list` command lists all
agents and daemons started by the user, while the `sudo launchctl list` lists all agents and daemons started by the
system.

For example, to find the identifier of the process with PID `62303`, run:

```shell
( /usr/bin/sudo launchctl list; launchctl list ) | grep 62303
```

The output will show the identifier, such as:

```
62303   0  com.fleetdm.orbit
```

You can now look in the standard locations for the `com.fleetdm.orbit.plist` file. Alternatively, you can use the
`launchctl dumpstate` command to dump the state of all launchd jobs, including the `.plist` files that started them. For
example, in a macOS system with Fleet's orbit running, you can run:

```shell
launchctl dumpstate | grep -B 1 -A 4 -E "active count = [1-9]" | grep com.fleetdm.orbit
```

And the output will show the path to the `.plist` file:

```shell
system/com.fleetdm.orbit = {
    path = /Library/LaunchDaemons/com.fleetdm.orbit.plist
```

You can now view the contents of the `.plist` file to understand how the process was started.

```shell
plutil -p /Library/LaunchDaemons/com.fleetdm.orbit.plist
```

## Create and edit .plist files with PlistBuddy

`PlistBuddy` is a powerful built-in macOS tool for creating and editing `.plist` files. You can use it to automate the
creation and modification of launchd agents and daemons.

You can create and edit `.plist` files using the `PlistBuddy`. For example, to create a new `.plist` file with a
key-value pair, run:

```shell
/usr/libexec/PlistBuddy -c "Add :Label string com.fleetdm.orbit" com.fleetdm.orbit.plist
```

To edit an existing `.plist` file, use the `-c` flag with the `Set` command. For example, to change the above `Label`
key to `com.fleetdm.orbit2`, run:

```shell
/usr/libexec/PlistBuddy -c "Set :Label com.fleetdm.orbit2" com.fleetdm.orbit.plist
```

Run `/usr/libexec/PlistBuddy --help` for more information on using `PlistBuddy`.

## Further reading

- Recently, we showed [two ways to turn a script into a macOS install package](../script-only-macos-install-package/).
- Previously, we explained [how to configure mTLS using the macOS keychain](../mtls-with-apple-keychain/).
- We also covered [how to create signed URLs with AWS CloudFront](../cloudfront-signed-urls/).

## Watch the video on launchd agents and daemons, and how to find the plist file for a running process

{{< youtube idFJmajURpE >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
