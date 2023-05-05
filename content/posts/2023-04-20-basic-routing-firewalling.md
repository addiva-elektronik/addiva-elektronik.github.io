---
title:  "Basic Routing and Firewalling"
date:   2023-04-20 10:23:40 +0200
author: Joachim Wiberg
draft:  false
categories:
 - networking course
---

We explore the concepts of basic IP routing, IPv4 only, and firewalling
with a focus on Linux.  Having read this, you should be able to answer
the following questions:

  - How is the routing table evaluated?
  - What happens to the TTL field in the IP header when a packet passes a router?
  - How is the firewall table evaluated?

<!-- more -->

## Routing Basics

The routing table is used for a lot of networking decisions, not just
for the "routing" of packets between two interfaces.  E.g., when you
send a PING message to an IP address, the routing table is consulted to
figure out the "next hop" address and the outbound interface.

```
~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.1     0.0.0.0         UG    600    0        0 wlan0
169.254.0.0     0.0.0.0         255.255.0.0     U     1000   0        0 virbr0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.1.0     0.0.0.0         255.255.255.0   U     600    0        0 wlan0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0
```

This classic UNIX `route` command shows the typical output of a routing
table on a Linux PC.  We can guess a few things about the system just by
looking at the `Iface` column: the system default route is via a WiFi
interface, the system has Docker running, and there is a virtual bridge
setup, probably to run Qemu using virt-manager.

As you can see, the routing table is sorted according to the netmask, or
prefix length.  The table is evaluated using *best prefix match*, which
means the default route using 192.168.1.1, that has a zero netmask, is
as it should be, the last resort for TCP/IP stack.  If nothing better
matches a destination IP address, the packet is sent to the default
gateway to deal with.

**Example 1:**

 - We want to send a PING to 192.168.1.42
 - The TCP/IP stack takes the netmask for each entry, masks 192.168.1.42
   with it and compares to the `Destination` field
 - Here it obviously matches with 192.168.1.0, on wlan0

This particular example is simple and should be fairly obvious even to a
beginner.  But what if we have a non-obvious routing table, like this:

```
~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
172.16.0.0      0.0.0.0         255.240.0.0     U     0      0        0 eth1
172.16.0.0      0.0.0.0         255.255.192.0   U     0      0        0 eth2
172.16.0.0      0.0.0.0         255.240.255.192 U     0      0        0 eth3
```

**Example 2:**

 - We want to send a PING to 172.16.0.10
 - The same process takes place as in the first example
 - *Best prefix match* elects eth3 as outbound interface for the PING

The *best prefix match* is the *longest prefix* match.  After that, the
last matching entry if there are multiple matching entries with the same
prefix.  When this happens there is usually something wrong with the
network setup, e.g., the following is wrong:

```
~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
192.168.1.0     0.0.0.0         255.255.255.0   U     600    0        0 eth0
192.168.1.0     0.0.0.0         255.255.255.0   U     600    0        0 eth1
```

### Static Routing

There are many ways to populate the routing table.  When you set an IP
address on an interface, the kernel automatically adds a "connected
route" for that interface -- meaning, the system has a direct connection
to that subnet.

The default route is usually set statically at boot, or static:ish by
a DHCP client.  The latter is probably the most common.

TBD

### Dynamic Routing

TBD

## Firewalling

Managing a firewall is specific to the operating system used.  On Linux
the kernel firewall is called *Netfilter*, it has been managed by many
different tools over the years, most recently the `iptables` and `nft`
userspace commands, where the former is the oldest of the two.

TBD
