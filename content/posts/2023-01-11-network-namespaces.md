---
title:  Network Namespaces
author: Joachim Wiberg
date:   2023-01-11 15:40:42 +0100
---

This time we show how you can easily use Linux *network namespaces* to
isolate one or more physical (or virtual) network interfaces from the
rest of your system.  Super useful when debugging network problems or
developing networking applications.

<!-- more -->

> Note: all commands here may require the use of `sudo`, unless you have
> [capabilities](https://troglobit.com/2016/12/11/a-life-without-sudo/)
> enabled for your user, for certain commands.  Try to avoid running as
> root, it can be avoided and saves you from a lot of headache.


Setup
-----

First, create a network namespace, call it what you like:

    ip netns add lab

Now move your interface to it.  The interface will be completely removed
from the "root" network namespace:

    ip link set eth1 netns lab

> **Note:** to move your interface back to the "root" network namespace
> from `lab`, use `ip netns exec world ip link set eth1 netns 1`

Bring up basic networking inside the namespace, remember: the loopback
device is very important in Linux:

    ip netns exec lab ip addr add 127.0.0.1/8 dev lo
    ip netns exec lab ip link set lo up

I like to consider the netns as an end-device, so I usually rename the
interface, which makes scripting after this point easier as well:

    ip netns exec lab ip link set eth1 down
    ip netns exec lab ip link set eth1 name eth0

You can also, optionally, add one or more VLAN interface(s) here, useful
when you want to debug VLAN trunks.  Remember, if you add a VLAN in this
step, make sure to use the `eth0.1` name in all the commands below this:

    ip netns exec lab ip link add link eth0 name eth0.1 type vlan id 1

At this point we can bring the interface(s) up and set IP address and
routes on them, like this:

    ip netns exec lab ip link set eth1 up
    ip netns exec lab ip addr add 192.168.2.42/24 dev eth1

    ip netns exec lab ip rout add 192.168.2.0/24 dev eth1
    ip netns exec lab ip rout add default via 192.168.2.1   # optional!

It's starting to get quite tedious to type `ip netns exec lab ...` all
the time.  So let's start a shell inside the namespace, it's almost like
a proper container:

    ip netns exec lab bash

Type `ifconfig` or `ip link` and `ip addr` to see your setup.  You are
completely isolated now from all the other network interfaces on your
system.  From here on you can run [ping and tcpdump][1] freely without
having to worry about your ping packets not properly egressing the right
interface, or not catching all traffic that's actually on the wire.


Concluding Remarks
------------------

When you exit your shell session and leave the network namespace, you
may want to clean up and restore your interface back to the system:

    # Restore original name (avoid clashing when moving it back)
    ip netns exec lab ip link set eth0 down
    ip netns exec lab ip link set eth0 name eth1
    ip netns exec lab ip link set eth1 up

    # Remove the network namespace (loopback removed automatically)
    ip netns del lab

Now, as an exercise, put all these commands in a script so you can more
easily and safely repeat this every time you need it.  I have a script
called `~/bin/usenet`, which I call with `sudo usenet eth1` :-)

[1]: {{< ref 2023-01-11-ping-and-tcpdump >}}
