---
title:  "Bridging and VLANs"
author: Joachim Wiberg
date:   2023-01-11 15:07:42 +0100
categories:
 - networking course
---

Moving on to more advanced concepts, this post shows how to set up
bridging (switching) and VLANs in Linux.  This overview should be
enough for you to elaborate on:

  - What is bridge?
  - What is the difference between a port and an interface?
  - What is a VLAN and how does it relate to a bridge?

<!-- more -->

## Bridging, or Switching

The [first post]({{< ref 2023-01-11-basic-networking >}}) introduced
LANs and broadcast domains.  An Ethernet bridge, or more commonly, a
switch, connects multiple networks segments into a common broadcast
domain.  For more information, see [the Wikipedia page on bridging][1].

In Linux we can create a software defined switch by adding multiple
network interfaces (NICs) to a PC and then connect them to the bridge
module.  In this setup these interfaces are called ports, and we don't
set IP addresses on them.  Instead, we do that on the bridge, and on
interfaces on top of the bridge.

                        br0          <---- bridge interface
                     ____|____
                    |#_#_#_#_#|      <---- bridge
                    /  |   |  \
                eth0 eth1 eth2 eth3  <---- ports


## Virtual LANs, VLANs

To kick things up a notch we need to introduce one more concept before
moving on -- [VLANs][2]!

A VLAN, or *virtual* LAN, is one of the true corner stones in most
network setups, and as such is really deserves a blog post of its own.

However, for the purpose of this post, consider VLANs a way for us to
*group ports* in separate broadcast domains.  I.e., isolate certain end
devices from each other; e.g., an office network from a process control
network.

                          br0
                     ______|______
                    |#_#_#_#_#_#_#|
                    /  |   :  |    \ 
                eth0 eth1  :  eth2 eth3
                           :
                  VLAN 10  :    VLAN 20

Here we have configured the bridge (switch) to assign ports eth0 and
eth1 to VLAN 10, and eth2 and eth3 to VLAN 20.  Ports in each VLAN can
only communicate with each other, the bridge ensures a true separation
between both VLANs.

If a device on port eth0 (member of VLAN 10) wants to communicate with a
device on port eth3 (member of VLAN 20) it must be routed somehow.  For
this to work we must either connect a router to ports eth1 and eth2, or
let interface br0 be a member of both VLANs.

> A port that is member of more than one VLAN is often referred to as a
> *trunk port*, and a port facing an end device is called *access port*.

Port VLAN memberships can be *tagged* or *untagged*.  A tagged port is
usually a trunk port, and an untagged port is usually an access port.
There are always exceptions to these rules, but for most cases this is a
good starting point.

To route traffic between VLAN 10 and VLAN 20 we create the following
setup (it's starting to look a bit crazy now):

    IP: 192.168.1.1                     IP: 192.168.2.1
                      br0.1     br0.2
                           \   /
                            br0
                       ______|______
                      |#_#_#_#_#_#_#|
                      /  |   :  |    \ 
                  eth0 eth1  :  eth2 eth3
                             :
                    VLAN 10  :    VLAN 20

Since `br0` now is a tagged member of both VLANs we need to create VLAN
interfaces on top of it to be able to set IP addresses.  These are the
gateway addresses each end device will use in their IP network setup.

That is basically it, remember to enable IP forwarding ... now let's get
hands-on with the command line!

> In the next section we use the names `vlan10` and `vlan20` instead of
> `br0.1` and `br0.2`, respectively.  On systems with multiple bridges
> this is not recommended, but otherwise it can greatly simplify for
> an end-user of the system.


## Creating a Bridge in Linux

There are actually two variants of the standard bridge in mainline
Linux; old-style and new-style.  The latter, which we will focus on in
this blog post, has native support for VLAN filtering.

    # ip link add br0 type bridge
    # ip link set br0 type bridge vlan_filtering 1

> **Note:** recent versions of Debian based systems, like Ubuntu, have
> enabled bridge firewalling by default.  This may completely disable
> all or some forwarding of traffic on bridges.  Causing a lot of head
> scratching!  See [Bridge Firewalling][] for a fix!

Now, add a couple of ports to the bridge:

    # ip link set eth0 master br0
    # ip link set eth1 master br0

To see the ports we use the [bridge(8)][] command, which is also part
of the iproute2 tool suite:

    # bridge link
    2: eth0: <BROADCAST,MULTICAST> mtu 1500 master br0 state disabled priority 32 cost 100 
    4: eth1: <BROADCAST,MULTICAST> mtu 1500 master br0 state disabled priority 32 cost 4 

To see the default VLAN assignments of ports:

    # bridge vlan show
    port    vlan ids
    eth0     1 PVID Egress Untagged
    eth1     1 PVID Egress Untagged
    br0      1 PVID Egress Untagged

So these ports look OK, the default VLAN ID assigned to ports is 1.
However, if the bridge would be created using command below (including option `vlan_default_pvid`):

    # ip link add br0 type bridge vlan_filtering 1 vlan_default_pvid 0

no default vlan id would be present. Thus the table would appear empty.

Let's add corresponding VLAN IDs for all of the PCs according to the previous figure.
We also set the `pvid` and `untagged` flags since we want to treat these ports as access ports (untagged).
Remember to remove from their default VLAN (ID 1) as well, if `vlan_default_pvid 0` was not used:

    # ip link set eth2 master br0
    # ip link set eth3 master br0
    # bridge vlan add vid 10 dev eth0 pvid untagged
    # bridge vlan add vid 10 dev eth1 pvid untagged
    # bridge vlan add vid 20 dev eth2 pvid untagged
    # bridge vlan add vid 20 dev eth3 pvid untagged
    # bridge vlan del vid 1 dev eth0
    # bridge vlan del vid 1 dev eth1
    # bridge vlan del vid 1 dev eth2
    # bridge vlan del vid 1 dev eth3

To see static and learned MAC addresses (c.f. the `arp` command):

    # bridge fdb show
    00:80:e1:42:55:a3 dev eth0 vlan 10 master br0 permanent
    00:80:e1:42:55:a3 dev eth0 master br0 permanent
    33:33:00:00:00:01 dev eth0 self permanent
    00:e0:4c:68:03:06 dev eth1 vlan 10 master br0 permanent
    00:e0:4c:68:03:06 dev eth1 master br0 permanent
    33:33:00:00:00:01 dev eth1 self permanent
	...
    33:33:00:00:00:01 dev br0 self permanent

In our use-case we have two different VLANs, so we need to change the
bridge port itself to be a *tagged* VLAN member, otherwise we cannot
distinguish between frames on different VLANs and thus cannot set up
our VLAN interfaces on top, like this:

        vlan10    vlan20     Layer-3 :: IP Networking
             \   /           -------------------------------
              br0
         ______|_______      Layer-2 :: Switching
        [#_#_#_#_#_#_#] 
        /  |   :  |    \     -------------------------------
    eth0 eth1  : eth2 eth3   Layer-1 :: Link layer
               :
      VLAN 10  :    VLAN 20

Let's change br0 to be a tagged member of VLAN 10 and 20:

    # bridge vlan add vid 10 dev br0 self
    # bridge vlan add vid 20 dev br0 self
    # bridge vlan show
    port    vlan ids
    eth0     10 PVID Egress Untagged
    eth1     10 PVID Egress Untagged
    eth2     20 PVID Egress Untagged
    eth3     20 PVID Egress Untagged
    br0      10
             20

Now we add our VLAN interface on top of `br0` so we can communicate with
the outside world.  Some prefer naming VLAN interfaces `br0.1`, but here
we use `vlan10` since we will only use one bridge:

    # ip link add name vlan10 link br0 type vlan id 10
    # ip addr add 192.168.1.1/24 dev vlan10
    # ip link add name vlan20 link br0 type vlan id 20
    # ip addr add 192.168.2.1/24 dev vlan20

Bring everything up by taking up the bridge and its ports:

    # ip link set eth0 up
    # ip link set eth1 up
    # ip link set eth2 up
    # ip link set eth3 up
    # ip link set br0 up
    # ip link set vlan10 up
    # ip link set vlan20 up

This is a good time to have a look at the available interfaces:

	# ip -brief link show
    lo          UNKNOWN  00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
    eth0        UP       00:80:e1:42:55:a0 <NO-CARRIER,BROADCAST,MULTICAST,UP>
    eth1        UP       00:80:e1:42:55:a1 <BROADCAST,MULTICAST,UP,LOWER_UP>
    eth2        UP       00:80:e1:42:55:a2 <NO-CARRIER,BROADCAST,MULTICAST,UP>
    eth3        UP       00:80:e1:42:55:a3 <BROADCAST,MULTICAST,UP,LOWER_UP>
    br0         UP       00:80:e1:42:55:a0 <BROADCAST,MULTICAST,UP,LOWER_UP>
    vlan10@br0  UP       00:80:e1:42:55:a0 <BROADCAST,MULTICAST,UP,LOWER_UP> 
    vlan20@br0  UP       00:80:e1:42:55:a0 <BROADCAST,MULTICAST,UP,LOWER_UP>

As you can see, the `vlan10` interface is created on top of `br0`,
`vlan10@br0`.  The addresses of all interfaces can be inspected with the
`ip address` command.  For a quick overview, use the `-brief` switch:

    # ip -br addr show
    lo               UNKNOWN        127.0.0.1/8
    eth0             UP             
    eth1             UP             
    eth2             UP             
    eth3             UP             
    br0              UP             
    vlan10@br0       UP             192.168.1.1/24
    vlan20@br0       UP             192.168.2.1/24

Here we have automatically configured IPv6 addresses on eth1 and br0,
this should be disabled since IP addresses in a our bridge setup should
only be set on the VLAN interfaces.


## Summary

In this post we covered the theory of Ethernet bridges and VLANs, and
then proceeded to provide an example of how to set this up a single
bridge up in Linux.

But wait, what if we want to connect two separate bridges, on two PCs,
with multiple VLANs on each?  Let's extend the image used previously,
and add a syntax for denoting VLAN memberships: *10U* means untagged
member of VLAN 10, *20U* means untagged in VLAN 20, and *10T* means tagged
member of VLAN 10, etc.

      vlan10    vlan20              vlan10     vlan20
           \   /                        \   /
            br0  10T,20T                 br0  10T,20T,30T
         ____|____                    ____|__________
        [#_#_#_#_#]                  [#_#_#_#_#_#_#_#]
        /  |      \                  /  |   |   \    \
    eth2  eth1     eth0----------eth0 eth1 eth2  eth3 eth4
     20U   10U           10T,20T       10U  20U   30U  30U

The image shows two devices with one bridge each.  The right-hand bridge
has more ports and VLANs, but they are interconnected using port eth0 on
each bridge.  This shared link, VLAN "trunk" (see above), serves as the
backbone for this network.

> Notice how VLAN 30 only exists on the right-hand bridge, both bridges
> filter traffic going out and coming in on the trunk from port eth0, to
> prevent VLAN 30 from reaching beyond its boundary (port eth3 and eth4).

[1]: https://en.wikipedia.org/wiki/Bridging_(networking)
[2]: https://en.wikipedia.org/wiki/IEEE_802.1Q
[bridge(8)]: http://man7.org/linux/man-pages/man8/bridge.8.html
[Bridge Firewalling]: /2023/03/29/bridge-firewalling/
