---
title:  Network Troubleshooting (1/N)
author: Joachim Wiberg
date:   2023-01-11 15:22:42 +0100
tags:
 - ping
 - tcpdump
---

Time for troubleshooting!  In this post we show how you can localize
issues in just about any network setup with:
[ping(8)](https://www.man7.org/linux/man-pages/man8/ping.8.html) and
[tcpdump(8)](https://www.man7.org/linux/man-pages/man8/tcpdump.8.html).


Ping can generate unicast, multicast, and broadcast traffic.  While
tcpdump is capable of capturing, formatting, and, most importantly,
*filtering* all types of traffic.

<!-- more -->

Recall the last picture from [Bridging and VLANs][]:

```
  vlan10    vlan20              vlan10   vlan20
       \   /                        \   /
        br0  10T,20T                 br0  10T,20T,30T
     ____|____                    ____|__________
    [#_#_#_#_#]                  [#_#_#_#_#_#_#_#]
    /  |      \                  /  |   |   \    \
eth2  eth1     eth0----------eth0 eth1 eth2  eth3 eth4
 20U   10U           10T,20T       10U  20U   30U  30U
 |     |                           |    |     |    |
 |     |                           |    |     |    |
eth0  eth0                        eth0 eth0  eth0 eth0
ED1   ED2                         ED3  ED4   ED5  ED6
```
*Figure 1: All end-devices (ED) have a single interface `eth0`*

Much can go wrong setting that up.  Not just with the standard Linux
bridge, but also with any HW offloading switch, and their respective
drivers.   To test the setup we can use our friends ping and tcpdump.
The following subnets and IP addresses are used:

**Subnets:**

  - VLAN 10: 192.168.1.0/24
  - VLAN 20: 192.168.2.0/24

**Left:**

  - vlan10: 192.168.1.1
  - vlan20: 192.168.2.1
  - ED1: 192.168.2.11
  - ED2: 192.168.1.11

**Right:**

  - vlan10: 192.168.1.2
  - vlan20: 192.168.2.2
  - ED3: 192.168.1.22
  - ED4: 192.168.2.22
  - ED5: 192.168.3.33
  - ED6: 192.168.3.34


Connectivity Between 2.11 and 2.22
----------------------------------

We start by verifying connectivity between ED1 on the left and ED4 on
the right.  They should both be untagged members in VLAN 20, and the VLAN
trunk between the bridges should carry the same VLAN tagged.  From ED1:

    root@ed1:~# ping 192.168.2.22

If we don't get a reply we can check with tcpdump on ED4:

    root@ed4:~# tcpdump -lni eth0 icmp

Dead silence.  So we go back to ED1 and change to a broadcast ping, this
should reach everyone connected to VLAN 20.  We check this with tcpdump
on all other ports.  First we check to see if we see anything on the
left bridge's VLAN 20:

    root@ed1:~# ping -b 192.168.2.255
    root@left:~# tcpdump -lni vlan20 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on vlan20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    08:49:32.306657 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 54, length 64
    08:49:33.330651 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 55, length 64
    08:49:34.354554 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 56, length 64
    08:49:35.378632 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 57, length 64

Here we see the ICMP traffic, so we can move on to check if we get the
same traffic on the right-hand bridge as well:

    root@right:~# tcpdump -lni vlan20 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on vlan20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    08:51:58.738186 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 197, length 64
    08:51:59.762284 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 198, length 64
    08:52:00.786212 IP 192.168.2.11 > 192.168.2.255: ICMP echo request, id 5, seq 199, length 64

Yup, so what's wrong here?  Verify the VLAN membership on the bridge:

    root@right:~# bridge vlan
    port              vlan-id
    eth4              30 PVID Egress Untagged
    eth3              30 PVID Egress Untagged
    eth2              10 PVID Egress Untagged
    eth1              20 PVID Egress Untagged
    eth0              10
                      20
    br0               10
                      20

There we have it!  Ports `eth1` and `eth2` had been mixed up in their
VLAN assignments!


Deep Dive in the Stack
----------------------

Now that we've covered a basic troubleshooting case, let's dive into the
various layers in the networking stack of one of the bridging devices.

```
    IP: 192.168.1.1   vlan10    vlan20  IP: 192.168.2.1
                           \   /
                            br0
                       ______|______
                      |#_#_#_#_#_#_#|
                      /  |   :  |    \ 
                  eth1 eth2  :  eth3 eth4
                             :
                    VLAN 10  :   VLAN 20
```

We use the same basic tools, inject ICMP traffic with ping on one port
and use tcpdump to see where it ends up.  Here we'll use broadcast from
an "end-device" attached to eth0:

```
    IP: 192.168.1.1   vlan10    vlan20   IP: 192.168.2.1
                           \   /
                            br0
NS1: 192.168.1.10      ______|______
--------.             |#_#_#_#_#_#_#|
lo      :             /  |   :  |    \ 
eth0    :         eth1 eth2  :  eth3 eth4
    `-------------'          :
--------'           VLAN 10  :   VLAN 20
```

The "end-device" is a network namespace on a dedicated device, with a
dedicated network card, or a VETH pair with one end in the namespace,
and the other attached to the bridge.  The latter is useful for testing
on the same device where the bridge runs.

    root@ns1:~# ping -b 192.168.1.255

On the system itself we can start by running tcpdump at the bottom, the
interface connected to the bridge.  This should work regardless if the
system has bridging (switch) offloading to an underlying hardware, since
broadcast is forwarded to all hosts on a LAN.

    root@system:~# tcpdump -lni eth1 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on eth3, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    09:05:26.671368 IP 192.168.1.10 > 192.168.1.255: ICMP echo request, id 5, seq 986, length 64
    09:05:27.695401 IP 192.168.1.10 > 192.168.1.255: ICMP echo request, id 5, seq 987, length 64
    09:05:28.719354 IP 192.168.1.10 > 192.168.1.255: ICMP echo request, id 5, seq 988, length 64

Here we can see basic ICMP frames, nothing strange, but we know there's
more going on here, so we add `-e` to get the Ethernet header as well:

    root@system:~# tcpdump -elni eth1 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    09:07:01.903045 3c:18:a0:07:32:46 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.1.255: ICMP echo request, id 5, seq 1079, length 64
    09:07:02.927090 3c:18:a0:07:32:46 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.1.255: ICMP echo request, id 5, seq 1080, length 64
    09:07:03.951079 3c:18:a0:07:32:46 > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 98: 192.168.1.10 > 192.168.1.255: ICMP echo request, id 5, seq 1081, length 64

Nothing much interesting still, just a standard Ethernet header.
Destination address ff:ff:ff:ff:ff:ff is the broadcast address,
and 3c:18:a0:07:32:46 is the source MAC address of eth0 in the
NS1 namespace.  The type/len field is 0x0800, which tcpdump has
translated to IPv4 for us.

OK, so port `eth1` is attached to the bridge, let's check the
bridge interface to see if there's anything interesting going
on there, again with the `-e` option:

    root@system:~# tcpdump -elni br0 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    09:08:25.870765 3c:18:a0:07:32:46 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 102: vlan 10, p 0, ethertype IPv4 (0x0800), 192.168.2.42 > 192.168.2.255: ICMP echo request, id 5, seq 1161, length 64
    09:08:26.894762 3c:18:a0:07:32:46 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 102: vlan 10, p 0, ethertype IPv4 (0x0800), 192.168.2.42 > 192.168.2.255: ICMP echo request, id 5, seq 1162, length 64
    09:08:27.918788 3c:18:a0:07:32:46 > ff:ff:ff:ff:ff:ff, ethertype 802.1Q (0x8100), length 102: vlan 10, p 0, ethertype IPv4 (0x0800), 192.168.2.42 > 192.168.2.255: ICMP echo request, id 5, seq 1163, length 64

Finally, something interesting!  On ingress the bridge has marked all
ICMP frames from NS1 as member of VLAN 10.  This is called a VLAN "tag"
and all Ethernet frames inside the bridge has one (in a bridge with
`vlan_filtering` enabled).  It is used to determine which ports in
a bridge that are eligible to receive the frame, only our tagged and
untagged ports in VLAN 10 in this case.  Since this is broadcast, all
ingressing frames will egress all other ports in VLAN 10.

If we change to ping the unicast address of the `vlan10` interface on
`br0`, a few other interesting things emerge:

    root@ns1:~# ping 192.168.1.1
    PING 192.168.1.10 (192.168.2.122) 56(84) bytes of data.
    64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=1.08 ms

First of all, we get a reply!  By default, Linux does not reply to
broadcast pings (configurable, but outside the scope of this blog, see
[sysctl(8)](https://www.man7.org/linux/man-pages/man8/sysctl.8.html)).
The output from tcpdump on our system tells a similar story, only the
destination MAC address is now first the MAC address of the `vlan10`
interface, and then in the reply, the MAC address of our `eth0` in
NS1.

    root@system:~# tcpdump -elni br0 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    09:20:26.272791 3c:18:a0:07:32:46 > 0e:de:c5:a9:cb:01, ethertype 802.1Q (0x8100), length 102: vlan 10, p 0, ethertype IPv4 (0x0800), 192.168.2.42 > 192.168.2.122: ICMP echo request, id 6, seq 6, length 64
    09:20:26.273073 0e:de:c5:a9:cb:01 > 3c:18:a0:07:32:46, ethertype 802.1Q (0x8100), length 102: vlan 10, p 0, ethertype IPv4 (0x0800), 192.168.2.122 > 192.168.2.42: ICMP echo reply, id 6, seq 6, length 64

The bridge has `learning` enabled and figures out the port to forward
the reply on using its forwarding database (FDB), let's inspect it!

> The FDB is for unicast and the MDB is for multicast addresses. On
> some physical switchcore fabrics this is a single entity, sometimes
> referred to as Address Translation Unit (ATU)

    root@system:~# bridge fdb show
    ...
    3c:18:a0:07:32:46 dev eth1 vlan 10 master br0
    ...

There's a lot of output, but the interesting line is the one matching
the MAC address we're looking for.  This line tells us the end station
is downstream on port `eth1`, and we learned this on VLAN 10 and on
bridge `br0`.  So we check with tcpdump that we only see the reply
on port `eth1`:

    root@basis:~# tcpdump -elni eth1 icmp
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    09:32:27.118368 3c:18:a0:07:32:46 > 0e:de:c5:a9:cb:01, ethertype IPv4 (0x0800), length 98: 192.168.2.42 > 192.168.2.122: ICMP echo request, id 7, seq 1, length 64
    09:32:27.118739 0e:de:c5:a9:cb:01 > 3c:18:a0:07:32:46, ethertype IPv4 (0x0800), length 98: 192.168.2.122 > 192.168.2.42: ICMP echo reply, id 7, seq 1, length 64

Yup, only on `eth1` and we also see that the bridge has stripped the
VLAN tag from the frame on egreess to the wire.  Remember, port `eth1`
is an *untagged* member of VLAN 10 on `br0`.


[ping(8)]: https://www.man7.org/linux/man-pages/man8/ping.8.html
[tcpdump(8)]: https://www.man7.org/linux/man-pages/man8/tcpdump.8.html
[sysctl(8)]: https://www.man7.org/linux/man-pages/man8/sysctl.8.html
[Bridging and VLANs]: {% post_url 2023-01-11-bridging-and-vlans %}
