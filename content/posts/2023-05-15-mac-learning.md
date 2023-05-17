---
title: "MAC Learning Basics"
date: 2023-05-15 09:03:20 +0200
draft: false
---

This post illustrates _dynamic learning of unicast MAC addresses_ on
Linux MAC Bridges, and how to use the `bridge fdb show` command to
view the MAC database. It expands upon an [earlier
post][Bridging-VLAN-Post] on Bridging and VLANs.


If you wish to try this out yourself, there are hints on exploring _MAC
Learning with Infix and GNS3_ at [the
bottom](#exploring-mac-learning-with-infix) of this post.

**\#learning-cache \#mac-database \#forwarding-database \#fdb
\#mac-bridge \#transparent-bridge**



## Example Topology - A single bridge

The figure below shows a sample topology with a bridge (Bridge-A) and
two connected hosts.  H1 and H2 have IP addresses on the same
10.0.1.0/24 subnet. The MAC addresses of H1, H2 and Bridge-A have been
selected for the purpose of easy illustration only. In typical
scenarios, these are [assigned by the hardware
manufacturer][wiki-mac].

```
   Bridge-A BaseMAC
   02:aa:aa:aa:00:00
   .----------------------.
   |        Bridge-A      |
   | eth0    eth1    eth2 |
   '--+-------+-------+---'
      |       |       |
      |       | 
      |     .-+--.
      |     | H2 |
      |     '----'
   .--+-.   IP: 10.0.1.2/24
   | H1 |  MAC: 02:22:22:00:00
   '----'
 IP: 10.0.1.1/24
MAC: 02:11:11:11:00:00
```
The following commands can be used to setup Bridge-A. (For more
details, see [Exploring MAC learning with 
Infix](#exploring-mac-learning-with-infix) further below.)

```
root@bridge-a:~# ip link add br0 type bridge
root@bridge-a:~# ip link set eth0 master br0
root@bridge-a:~# ip link set eth1 master br0
root@bridge-a:~# ip link set eth2 master br0
root@bridge-a:~# ip link set br0 up
```

### Bridges forward packets based on destination MAC ...


Bridges look at the destination MAC when deciding onto what port a
packet should be sent. It keeps information it its MAC database (a.k.a
_forwarding database_, _FDB_, _learning_cache_, _station cache_, etc.). If the
address is found, it forwards the packet onto that port. If the
address is not found, it floods the packet onto all ports.

```
root@bridge-a:~# bridge fdb show dynamic
02:11:11:11:00:00 dev eth0 master br0 
02:22:22:22:00:00 dev eth1 master br0 
root@bridge-a:~# 
```

For Bridge-A, packets to _02:11:11:11:00:00_ are forwarded to eth0,
packets to 02:22:22:22:00:00 to eth1, and packet to other addresses
are forwarded to eth0, eth1, and all other ports of the bridge.

### ... and learns addresses based on source MAC

When the bridge starts, the _MAC database_ is empty, thus all traffic
is flooded. For every packet received, the bridge learns the location
of the sender by inspecting the source MAC of the packet.

```
root@bridge-a:~# bridge fdb show dynamic
root@bridge-a:~# 
```
Once H1, sends a packet, the bridge will learn that H1 resides on port
eth0. Or more precisely, that the MAC address of H1 resides on port
eth0.

```
root@bridge-a:~# bridge fdb show dynamic
02:11:11:11:00:00 dev eth0 master br0 
root@bridge-a:~# 
```

### Learned entries are aged out

Learned MAC entries are kept in the MAC database, but will eventually
be aged out if the entry is not refreshed. The aging time is typically
300 seconds (5 minutes), and entry is refreshed every time bridge
receives a packet with that MAC.

## Multiple Bridges

The MAC learning can be extended to multiple bridges. In the example
below, we have expanded the LAN by adding a second bridge.

```
   Bridge-A BaseMAC               Bridge-B BaseMAC
   02:aa:aa:aa:00:00              02:bb:bb:bb:00:00
   .----------------------.       .-----------------------.
   |        Bridge-A      |       |       Bridge-B        |
   | eth0    eth1    eth2 |       | eth0    eth2     eth2 |
   '--+-------+-------+---'       '--+-------+--------+---'
      |       |       |              |       |        |
      |       |       '--------------'       |        | 
      |     .-+--.                        .--+-.      |
      |     | H2 |                        | H3 |      |
      |     '----'                        '----'      |
   .--+-.  IP: 10.0.1.2/24          IP: 10.0.1.3/24 .-+--.
   | H1 | MAC: 02:22:22:00:00  MAC: 02:33:33::00:00 | H4 |
   '----'                                           '----'
 IP: 10.0.1.1/24                            IP: 10.0.1.1/24
MAC: 02:11:11:11:00:00               MAC: 02:44:44:44::00:00
```

Inspecting the MAC databases at Bridge A and B after booting up, they
may show the following result
```
On Bridge A:
root@bridge-a:~# bridge fdb show dynamic
02:11:11:11:00:00 dev eth0 master br0 
02:22:22:22:00:00 dev eth1 master br0 
02:bb:bb:bb:00:00 dev eth2 master br0 
root@bridge-a:~# 

On Bridge B:
root@bridge-b:~# bridge fdb show dynamic
02:aa:aa:aa:00:02 dev eth0 master br0 
02:33:33:33:00:00 dev eth1 master br0 
02:44:44:44:00:00 dev eth2 master br0 
root@bridge-b:~# 
```

We can see that bridges know their local hosts, and have also learned
about each-other.  The reason MAC databases are not completely empty
here is that all these nodes exchange LLDP messages. The LLDP packets only go
hop-by-hop, thus Bridge-A will not learn MAC addresses of H3 and H4 this
way. For example, if we listen in on port eth0 of Bridge-A, we can see
the following LLDP traffic.
```
root@bridge-a:~# tcpdump -ee -n -i eth0
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
07:56:58.915592 02:11:11:11:00:00 > 01:80:c2:00:00:0e, ethertype LLDP (0x88cc), length 208: LLDP, length 194: h1
07:57:07.452643 02:aa:aa:aa:00:00 > 01:80:c2:00:00:0e, ethertype LLDP (0x88cc), length 200: LLDP, length 186: bridge-a
^C
2 packets captured
2 packets received by filter
0 packets dropped by kernel
root@bridge-a:~# 
```

Now, let H1 ping H2 (`ping 10.0.1.2`) and inspect the MAC databases on
the bridges: 
```
Bridge-A:
root@bridge-a:~# bridge fdb show dynamic
02:11:11:11:00:00 dev eth0 master br0 
02:22:22:22:00:00 dev eth1 master br0 
02:bb:bb:bb:00:00 dev eth2 master br0 
root@bridge-a:~# 

Bridge-B:
root@bridge-b:~# bridge fdb show dynamic
02:11:11:11:00:00 dev eth0 master br0   <====== NEW (H1)
02:aa:aa:aa:00:02 dev eth0 master br0 
02:33:33:33:00:00 dev eth1 master br0 
02:44:44:44:00:00 dev eth2 master br0 
root@bridge-b:~# 
```
The only thing that differs here is that Bridge-B has learned about H1
on its port eth0. It learns this from the **broadcast** `ARP Who Has
10.0.1.2?` sent by H1 (preceding the ICMP Echo Request of the Ping);
this broadcast ARP is flooded throughout the whole LAN. The 'ARP
Response' by H2 is sent by **unicast** and never reaches Bridge-B. (More
precisely, as Bridge-A knows where MAC 02:11:11:11:00:00 resides, it
forwards the 'ARP Response' directly on eth0.

Now, let H1 send a broadcast ping (`ping 10.0.1.255`). All hosts
respond.  If we again inspect the MAC databases at the bridges, we see
they have learned the location of the MACs for all hosts.

```
Bridge-A:
root@bridge-a:~# bridge fdb show dynamic
02:11:11:11:00:00 dev eth0 master br0 
02:22:22:22:00:00 dev eth1 master br0 
02:33:33:33:00:00 dev eth2 master br0    <=== NEW (H3)
02:44:44:44:00:00 dev eth2 master br0    <=== NEW (H4)
02:bb:bb:bb:00:00 dev eth2 master br0
root@bridge-a:~# 

Bridge-B:
root@bridge-b:~# bridge fdb show dynamic
02:22:22:22:00:00 dev eth0 master br0    <=== NEW (H2)
02:11:11:11:00:00 dev eth0 master br0 
02:aa:aa:aa:00:02 dev eth0 master br0 
02:33:33:33:00:00 dev eth1 master br0 
02:44:44:44:00:00 dev eth2 master br0 
root@bridge-b:~# 
```

The figure below illustrates the MAC databases (FDBs) by using
symbolic names for the MAC addresses.
```
   Bridge-A BaseMAC               Bridge-B BaseMAC
   02:aa:aa:aa:00:00              02:bb:bb:bb:00:00
   .----------------------.       .-----------------------.
   |        Bridge-A      |       |       Bridge-B        |
   | eth0    eth1    eth2 |       | eth0    eth2     eth2 |
   '--+-------+-------+---'       '--+-------+--------+---'
    h1|     h2|   br-b|              |br-a   |h3      |h4    <== FDB
      |       |     h3'--------------'h1     |        | 
      |     .-+--.  h4                h2  .--+-.      |
      |     | H2 |                        | H3 |      |
      |     '----'                        '----'      |
   .--+-.  IP: 10.0.1.2/24          IP: 10.0.1.3/24 .-+--.
   | H1 | MAC: 02:22:22:00:00  MAC: 02:33:33::00:00 | H4 |
   '----'                                           '----'
 IP: 10.0.1.1/24                            IP: 10.0.1.1/24
MAC: 02:11:11:11:00:00               MAC: 02:44:44:44::00:00
```

## Exploring MAC learning with Infix

The examples above were done using a virtual setup with
[Infix][Infix], specifically the Classic build, on [GNS3][GNS3]. Below
are some hints if you like to explore this yourself.

To install Infix and GNS3, look at the guide in the  [Infix][Infix]
README. It is also recommended to look at the [Basic Networking
Training Post][Training-Basic-Net-Post].

In the setup used here, all units (Bridge-A, Bridge-B, H1, H2, H3 and
H4) were using _Infix-Classic_ as GNS3 appliance.

### Hints on configuring the Bridges

Bridges-A and Bridges-B were configured with multiple ports (at
least 3). This is done in the _GNS3 Configure_ window for the units.

To make Bridge-A (and Bridge-B) act like MAC bridges, the following
commands where issued via the console.

```
root@bridge-a:~# ip link add br0 type bridge
root@bridge-a:~# ip link set eth0 master br0
root@bridge-a:~# ip link set eth1 master br0
root@bridge-a:~# ip link set eth2 master br0
root@bridge-a:~# ip link set br0 up
```

To make Bridge-A get hostname (and prompt) 'bridge-a', you can edit
'/etc/hostname'. To avoid that Bridge-A and Bridge-B sends a lot of
DHCP messages, you can comment out the associated line in
'/etc/network/interfaces.d/ for eth0, eth1, etc.

```
root@bridge-a:~# cat /etc/network/interfaces.d/eth0 
auto eth0
iface eth0 inet manual
#iface eth0 inet dhcp
#    pre-up ip link set eth0 group iface
root@bridge-a:~# 
```

The first line (auto eth0) could also be commented out, but then you
need to bring up eth0, eth1 and eth2 manually.
```
root@bridge-a:~# ip link set eth0 up
root@bridge-a:~# ip link set eth1 up
root@bridge-a:~# ip link set eth2 up
```


#### Hints on configuring the Hosts

To assign static IP 10.0.1.1/24 to H1 (etc.), update the
'/etc/network/interfaces.d/eth0'
```
root@h1:~# cat /etc/network/interfaces.d/eth0 
auto eth0
iface eth0 inet static
    address 10.0.1.1
    netmask 255.255.255.0
#iface eth0 inet dhcp
#    pre-up ip link set eth0 group iface
root@h1:~# 
```

To enable hosts respond to 'broadcast pings', change the corresponding
'proc' setting
```
root@h1:~# sysctl -w net.ipv4.icmp_echo_ignore_broadcasts=0
net.ipv4.icmp_echo_ignore_broadcasts = 0
root@h1:~#
```
To make the setting permanent, the following can be used:
```
root@h1:~# echo 'net/ipv4/icmp_echo_ignore_broadcasts = 0' > /etc/sysctl.d/91-icmp-echo-bcast.conf 
root@h1:~#
```

[GNS3]: https://docs.gns3.com/docs/
[Training-Basic-Net-Post]: ../../../../2023/01/11/network-training-basic-networking/index.html#install-gns3
[Bridging-VLAN-Post]: ../../../../2023/01/11/bridging-and-vlans/
[Infix]: https://github.com/kernelkit/infix
[wiki-mac]: https://en.wikipedia.org/wiki/MAC_address

