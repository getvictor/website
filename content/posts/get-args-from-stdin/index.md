+++
title = 'Why you should use STDIN to read your program arguments'
description = "Improve application security by reading sensitive program arguments from STDIN"
authors = ["Victor Lyuboslavsky"]
image = "stdin-headline.png"
date = 2024-08-12
categories = ["Software Development", "Security"]
tags = ["Golang", "Application Security"]
draft = false
+++

## STDIN is more secure than environment variables or command-line arguments

When you pass command-line arguments to a program, they are visible to anyone who can run the `ps` command. Allowing others to read arguments is a security risk if the arguments contain sensitive information like passwords or API keys.

Environment variables are also visible to anyone who can run the `ps` command. They are also globally visible to the program, so any arbitrary code in your application can extract the environment variables.

To get the environment variables of a process, run `ps eww <PID>`. For example:

```shell
$ ps eww 1710
    PID TTY      STAT   TIME COMMAND
   1710 pts/0    Ss+    0:00 bash SYSTEMD_EXEC_PID=1209 SSH_AUTH_SOCK=/run/user/1000/keyring/ssh SESSION_MANAGER=local/victor-ubuntu:@/tmp/.ICE-unix/1176,unix/victor-ubuntu:/tmp/.ICE-unix/1176 GNOME_TERMINAL_SCREEN=/org/gnome/Terminal/screen/ab0b9d6a_a699_4bc5_bb53_628be016afa5 LANG=en_US.UTF-8 XDG_CURRENT_DESKTOP=ubuntu:GNOME PWD=/home/victor WAYLAND_DISPLAY=wayland-0 DISPLAY=:0 QT_IM_MODULE=ibus USER=victor DESKTOP_SESSION=ubuntu XDG_MENU_PREFIX=gnome- HOME=/home/victor DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus SSH_AGENT_LAUNCHER=gnome-keyring _=/usr/bin/gnome-session XDG_CONFIG_DIRS=/etc/xdg/xdg-ubuntu:/etc/xdg VTE_VERSION=6800 XDG_SESSION_DESKTOP=ubuntu QT_ACCESSIBILITY=1 GNOME_DESKTOP_SESSION_ID=this-is-deprecated GNOME_SETUP_DISPLAY=:1 GTK_MODULES=gail:atk-bridge LOGNAME=victor GNOME_TERMINAL_SERVICE=:1.83 GNOME_SHELL_SESSION_MODE=ubuntu XDG_RUNTIME_DIR=/run/user/1000 XMODIFIERS=@im=ibus SHELL=/bin/bash XDG_SESSION_TYPE=wayland PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin USERNAME=victor COLORTERM=truecolor XAUTHORITY=/run/user/1000/.mutter-Xwaylandauth.R816R2 XDG_DATA_DIRS=/usr/share/ubuntu:/usr/local/share/:/usr/share/:/var/lib/snapd/desktop IM_CONFIG_PHASE=1 TERM=xterm-256color GDMSESSION=ubuntu XDG_SESSION_CLASS=user
```

STDIN is more secure because it is not visible to the `ps` command and is not globally visible to the program. Thus, only the parts of the program that explicitly read from STDIN can access this data.

## How to read program arguments from STDIN with Go

In the code example below, we check if any data is being piped in from STDIN with `os.ModeNamedPipe`. Then, we wait to read all the data from STDIN with `ioutil.ReadAll`. Finally, we parse the STDIN data just like a shell would using the [github.com/kballard/go-shellquote](https://github.com/kballard/go-shellquote) library and append it to any existing command-line arguments.

{{< gist getvictor c345f324ebf6dfa64df9a8c0919d6672 >}}

## How to integrate with a secret manager

One way to securely pass sensitive information to a program is to store it in a secret manager like [1Password](https://developer.1password.com/docs/cli/secret-references). Then, you can read the secret from the secret manager and pass it to the program via STDIN. For example:

```
echo --secret $(op read op://employee/example_server/secret) | go run read-args-from-stdin.go
```

## Further reading

Recently, we discussed how to [unmarshal JSON payloads with null, set, and missing keys using Go](../go-json-unmarshal/).

Previously, we wrote [how we catch missed authorization checks in our Go application](../catch-missed-authorization-checks-during-software-development).

## Watch how to read program arguments from STDIN

{{< youtube Jg7xItfa6t8 >}}

_Note:_ If you want to comment on this article, please do so on the YouTube video.
