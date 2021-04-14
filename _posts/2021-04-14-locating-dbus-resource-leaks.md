---
layout: post
caption: Locating D-Bus Resource Leaks
categories: [fedora]
tags: [dbus, dbus-broker, quota, resource, leak, accounting]
hidden: true
---

With _dbus-broker_ we have introduced the resource-accounting of _bus1_
into the D-Bus world. We believe it greatly improves and strengthens
the resource distribution of the D-Bus messages bus, and we have already found
a handful of resource leaks that way. However, it can be a daunting task to
solve resource exhaustion bugs, so I decided to describe the steps we took to
resolve a recent resource-leak in the openQA package.

A few days ago, Adam Williamson approached me [^1] with a bug in the openQA
package, where he saw the log stream filled with messages like:

    dbus-broker[<pid>]: Peer :1.<id> is being disconnected as it does not have the resources to receive a reply or unicast signal it expects.
    dbus-broker[<pid>]: UID <uid> exceeded its 'bytes' quota on UID <uid>.

This is the typical sign of a resource exhaustion in _dbus-broker_. When the
message broker generates or forwards messages to an individual client, it will
queue them as outgoing-messages and push them into the unix-socket of the
client. If this client does not dequeue messages, this queue might fill up. If
a limit is reached, something needs to be done. Since D-Bus is not a lossy
protocol, dropping messages is not an option. Instead, the message broker will
either refuse new incoming operations or disconnect a client. All resources are
accounted on UIDs, this means multiple clients of the same user will share the
same resource limits.

Depending on what message is sent, it is accounted either on the receiver or
sender. Furthermore, some messages can be refused by the broker, others
cannot. The exact rules are described in the wiki [^2].

In the case of openQA, the first step was to query the accounting information
of the running message broker:

    sudo dbus-send --system --dest=org.freedesktop.DBus --type=method_call --print-reply /org/freedesktop/DBus org.freedesktop.DBus.Debug.Stats.GetStats

(Replace `--system` with `--session` to query the session or user bus.)

While preferably this query is performed when the resource exhaustion happens,
it will often yield useful information under normal operation as well.
Resources are often consumed slowly, so the accumulation will still show up.

The output [^3] of this query shows a list of all D-Bus clients with their
accounting information. Furthermore, it lists all UIDs that have clients
connected to this message bus, again with all accounting information. The
challenge is to find suspicious entries in this huge data dump. The most
promising solution so far was to search for `"OutgoingBytes"` and check for
big numbers. This shows the number of bytes queued in the message broker for
a particular client. It is usually 0, since the kernel queues are big enough
to hold most normal messages. Even if it is not 0, it is usually just a couple
of KiB.

In this case, we checked for `"OutgoingBytes"`, and found:

    dict entry(
        string "OutgoingBytes"
        uint32 62173024
    )

62 MiB of messages are waiting to be delivered to that client. Expanding the
logs to show the surrounding block, we see:

    struct {
        string ":1.211366"
        array [
            dict entry(
                string "UnixUserID"
                variant                            uint32 991
            )
            dict entry(
                string "ProcessID"
                variant                            uint32 674968
            )
            [...]
        ]

        array [
            [...]
            dict entry(
                string "Matches"
                uint32 1
            )
            [...]
            dict entry(
                string "OutgoingBytes"
                uint32 62173024
            )
            [...]
        ]
    }

This tells us the PID `674968` of user `991` has roughly 62 MiB of data queued,
and it is likely not dequeuing the data. Furthermore, we see it has 1 message
filter (D-Bus match rule) installed. D-Bus message filters will cause matching
D-Bus signals to be delivered to a client. So a likely problem is that this
client keeps receiving signals, but does not dispatch its client socket.

We digged further, and the data dump includes more such clients. Matching back
the PIDs to processes via `ps auxf`, we found that each and every of those
suspicious entries was `/usr/bin/isotovideo: backend`. The code of this process
is part of the `os-autoinst` repository, in this case `qemu.pm`. A quick look
showed only a single use of D-Bus [^4]. At a first glance, this looks alright.
It creates a system-bus connection via the _Net::DBus_ perl module, dispatches
a method-call, and returns the result. However, we know this process has a
match-rule installed (assuming the dbus-broker logs are correct), so we checked
further and found that the _Net::DBus_ module always installs a match-rule on
`NameOwnerChanged`. Furthermore, it caches the system-bus connection in a
global variable, sharing it across users in the same code-base.

Long story short, the `os-autoinst` qemu module created a D-Bus connection
which was idle in the background and never dispatched by any code. However, the
connection has a match-rule installed, and the message broker kept sending
matching signals to that connection. This data accumulated and eventually
exceeded the resource quota of that client. A workaround was quickly provided,
and it will hopefully resolve this problem [^5].

Hopefully, this short recap will be helpful to debug other similar situations.
You are always welcome to message us on _bus1-devel@googlegroups_ or on the
dbus-broker GitHub issue tracker if you need help.

[^1]: [OpenSUSE Bug #90872](https://progress.opensuse.org/issues/90872)
[^2]: [D-Bus Broker Accounting](https://github.com/bus1/dbus-broker/wiki/Accounting)
[^3]: [OpenSUSE Bug Attachment](https://progress.opensuse.org/attachments/11201/dbusdebug.txt)
[^4]: [os-autoinst qemu.pm](https://github.com/os-autoinst/os-autoinst/blob/965960f534c93ef12f0978014d589b8b2be6e6d2/backend/qemu.pm#L138)
[^5]: [os-autoinst PR #1641](https://github.com/os-autoinst/os-autoinst/pull/1641)
