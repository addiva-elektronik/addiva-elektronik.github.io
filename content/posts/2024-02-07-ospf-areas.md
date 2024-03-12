---
title:  "Using OSPF Areas in Infix"
author: Jon-Olov Vatn
date:   2024-02-07 17:07:42 +0200
toc:    true
tags:
 - linux
 - networking
 - infix
 - ospf
 - nssa
 - stub
 - redistribute
---

In an earlier [blog post][infix-unnumbered-post] we covered basics on
configuring OSPF in Infix. The current post extends this by covering
use of OSPF *Areas* in Infix. With OSPF areas you can form a two-level
hierarchy, consisting of a backbone area (0.0.0.0) and multiple
sub-areas, e.g., 0.0.0.1, 0.0.0.2, etc.

For a small network, all OSPF routers can reside in one area. But if
the network increases in terms of routers and networks, or if
maintained by different organizations, dividing the OSPF network into
areas is useful.

> For information on scalability issues in OSPF and how OSPF areas can
> be used to mitigate this, please see Narbid Kocharians' excellent
> [YouTube talk][youtube-ospf-areas] on OSPF Area Types.

Infix currently supports three area types: *regular* areas, *stub*
areas, and *not-so-stubby* areas (NSSAs). Please see [Infix
documentation][infix-doc] for updated information on OSPF support.

## Overview of intended topology

In this blog we show how to configure OSPF areas in Infix, using a
small topology to illustrate the method. Examples will start by using
*regular* areas, and then see how NSSA (or stub) areas can be used to
limit routing state in the sub-areas.

<a name="fig1">**Figure 1**: Overview of Example Topology</a> 

```
                            (External Routes, Redistribute)
                            (Could be towards Internet)
       
                                   172.16.0.0/24 .-------. 172.16.1.0/24
                                      .----------+ R Ext +-------------
                                     /         .2'-------'
                                    /.1
                             .-----+-. 
     . . . . . . . . . . . . | ASBR  | . . . . . . . . . . . . 
   .  AREA 0        .--------+ BB-R3 +--------.                .
 .    (Backbone)    |        '-------'        |                  .
 .  Main IT Dept.   |        10.0.3.3         |                  .
 .                  |                         |                  .
 .         10.0.1.1 |                         | 10.0.2.2         .
 .              .---+---.                 .---+---.              .
   .            | BB-R1 +-----------------+ BB-R2 |            .
     . . . . . .|  ABR  | . . . . . . . . |  ABR  |. . . . . .
   .  AREA 43   '---+---'                 '---+---'            .
 .    Dept. X       |                         |                  .
.                   |                         |                    .
.         10.43.1.1 |                         | 10.43.2.2          .
.    .-.        .---+----.               .----+---.        .-.     .
.    |H|        | A43-R1 +---------------+ A43-R2 |        |H|     .
.    '+'        '-+-+----'               '----+-+-'        '+'     .
.     |       .1 /  |                         |  \ .1       |      .
.  ---+---------´   |                         |   `---------+--    .
.   10.43.1.0/24    |                         |   10.43.2.0/24     .
.                   |                         |                    .
.         10.43.3.3 |                         | 10.43.4.4          .
.    .-.        .---+----.               .----+---.        .-.     .
.    |H|        | A43-R3 +---------------+ A43-R4 |        |H|     .
.    '+'        '-+------'               '------+-'        '+'     .
.     |       .1 /                               \          |      .
.  ---+---------´                                 `---------+--    .
.   10.43.3.0/24                                  10.43.4.0/24     .
  .                                                              .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
```

## The sub-area

We start by configuring the OSPF routers at department X, which has
been assigned OSPF area id '43' (0.0.0.43). That is, we configure all
routers in area 43, except the area border routers (ABRs), which are
maintained by the main IT department.

<a name="fig2">**Figure 2**: Routers in Area 43 (except BB-R1 and BB-R2)</a> 

```
                 BB-R1                     BB-R2
 AREA 43           ^                         ^
                   |                         |
         10.43.1.1 |                         | 10.43.2.2
    .-.        .---+----.               .----+---.        .-.
    |H|        | A43-R1 +---------------+ A43-R2 |        |H|
    '+'        '-+-+----'               '----+-+-'        '+'
     |       .1 /  |                         |  \ .1       |
  ---+---------´   |                         |   `---------+--
   10.43.1.0/24    |                         |   10.43.2.0/24
                   |                         |
         10.43.3.3 |                         | 10.43.4.4
    .-.        .---+----.               .----+---.        .-.
    |H|        | A43-R3 +---------------+ A43-R4 |        |H|
    '+'        '-+------'               '------+-'        '+'
     |       .1 /                               \          |
  ---+---------´                                 `---------+--
   10.43.3.0/24                                  10.43.4.0/24
```

For OSPF configuration, we have assumed the following:

- OSPF unnumbered interfaces are used for links between routers
- Bidirectional Forwarding Detection (BFD) is used for fast link-down
  detection.

Using OSPF unnumbered interfaces makes the configuration easier and
the routing tables smaller. You can, of course, use regular /30
subnets or similar on the links between Routers. In particular, the
link towards the ABR may have to use regular subnets as the ABR may
not support unnumbered interfaces. Here we assume all routers are
Infix routers. (For more details on OSPF unnumbered interfaces in
Infix, see the previous [blog post][infix-unnumbered-post].) 

Router 1 in area 43 (A43-R1) is used as illustration. We start by
configuring a suitable hostname.

```
admin@infix-36-00-00:~$ cli
admin@infix-36-00-00:/> configure
admin@infix-36-00-00:/config/> set system hostname a43-r1
admin@infix-36-00-00:/config/> leave
admin@a43-r1:/> copy running-config startup-config
admin@a43-r1:/
```

> Remember to `copy running startup` to store your changes
> permanently after leaving configuration context. This will not be
> shown in later examples.

### IP address and forwarding settings on R1 (A43-R1)

As show in [figure 2](#fig2) R1 needs four Ethernet interfaces. 
To see what interfaces we have, use `show interfaces`.

```
admin@a43-r1:/> show interfaces
INTERFACE       PROTOCOL   STATE       DATA
eth0            ethernet   DOWN        0c:91:31:36:00:00
eth1            ethernet   DOWN        0c:91:31:36:00:01
eth2            ethernet   DOWN        0c:91:31:36:00:02
eth3            ethernet   DOWN        0c:91:31:36:00:03
eth4            ethernet   DOWN        0c:91:31:36:00:04
eth5            ethernet   DOWN        0c:91:31:36:00:05
lo              ethernet   UP          00:00:00:00:00:00
                ipv4                   127.0.0.1/8 (static)
                ipv6                   ::1/128 (static)
admin@a43-r1:/>
```

R1 and the other routers in this example has six Ethernet interfaces
named *eth0*..*eth5*. We use *eth0* towards the ABR, *eth1* and *eth2*
for links to other routers in area 43, and *eth3* for the local subnet
(10.43.1.0/24).

```
admin@a43-r1:/>
admin@a43-r1:/> configure
admin@a43-r1:/config/> set interface lo ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth0 ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth1 ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth2 ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth3 ipv4 address 10.43.1.1 prefix-length 24
admin@a43-r1:/config/> set interface eth0 ipv4 forwarding true
admin@a43-r1:/config/> set interface eth1 ipv4 forwarding true
admin@a43-r1:/config/> set interface eth2 ipv4 forwarding true
admin@a43-r1:/config/> set interface eth3 ipv4 forwarding true
admin@a43-r1:/config/> leave
admin@a43-r1:/>
```

Running `show interfaces` again would show the following:

```
admin@a43-r1:/> show interfaces
INTERFACE       PROTOCOL   STATE       DATA
eth0            ethernet   DOWN        0c:91:31:36:00:00
                ipv4                   10.43.1.1/32 (static)
eth1            ethernet   DOWN        0c:91:31:36:00:01
                ipv4                   10.43.1.1/32 (static)
eth2            ethernet   DOWN        0c:91:31:36:00:02
                ipv4                   10.43.1.1/32 (static)
eth3            ethernet   DOWN        0c:91:31:36:00:03
                ipv4                   10.43.1.1/24 (static)
eth4            ethernet   DOWN        0c:91:31:36:00:04
eth5            ethernet   DOWN        0c:91:31:36:00:05
lo              ethernet   UP          00:00:00:00:00:00
                ipv4                   127.0.0.1/8 (static)
                ipv4                   10.43.1.1/32 (static)
                ipv6                   ::1/128 (other)
admin@a43-r1:/>
```

### OSPF settings on R1

It is time to configure OSPF settings. 

- Enable OSPF on the relevant interfaces
- Mark all *inter-router interfaces* as OSPF point-to-point
  interfaces.  (This combined with /32 IP addresses will make them an
  OSPF unnumbered interfaces.)
- Enable BFD on all inter-router interfaces (for fast link-down
  detection and OSPF route convergence).
- We set eth3 as OSPF passive interface (as there is no router to talk
  to on that LAN)

```
admin@a43-r1:/>
admin@a43-r1:/> configure
admin@a43-r1:/config/> edit routing control-plane-protocol ospfv2 name default
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface lo enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth0 enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth1 enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth2 enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth3 enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth0 interface-type point-to-point
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth1 interface-type point-to-point
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth2 interface-type point-to-point
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth0 bfd enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth1 bfd enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth2 bfd enabled true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth3 passive true
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@a43-r1:/>
```

Now, repeat IP settings (previous section) and OSPF settings for
routers R2, R3 and R4 in area 43 (see [figure 2](#fig2)), and connect
them via their respective eth1 and eth2. As we use unnumbered
interfaces, it does not matter if R1 connects eth1 to R2 and eth2 to
R3 or vice versa.

When done, we can verify if routers are able to discover each other and
exchange routing information. E.g., on R1 we can run `show ospf
neighbor` to see if R1 can see R2 (10.43.2.2) and R3 (10.43.3.3).

```
admin@a43-r1:/> show ospf neighbor

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
10.43.2.2         1 Full/-          5m15s             34.389s 10.43.2.2       eth1:10.43.1.1                       0     0     0
10.43.3.3         1 Full/-          14.994s           35.066s 10.43.3.3       eth2:10.43.1.1                       1     0     0

admin@a43-r1:/>
```

The routes exchanged would look like follows (assuming interfaces on
the host subnets have not yet got link up).

```
admin@a43-r1:/> show ospf routes
============ OSPF network routing table ============
N    10.43.1.1/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.2.2/32          [10] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.3.3/32          [10] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.4.4/32          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2

============ OSPF router routing table =============

============ OSPF external routing table ===========


admin@a43-r1:/>
```

If we attach some PC or similar on *eth3* on each router, routers get
link up and will advertise reachability to the host subnets too.

```
admin@a43-r1:/> show ospf routes
============ OSPF network routing table ============
N    10.43.1.0/24          [10] area: 0.0.0.43
                           directly attached to eth3
N    10.43.1.1/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.2.0/24          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.2.2/32          [10] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.3.0/24          [20] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.3.3/32          [10] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.4.0/24          [30] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2
N    10.43.4.4/32          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2

============ OSPF router routing table =============

============ OSPF external routing table ===========


admin@a43-r1:/>
```

## Backbone routers


<a name="fig3">**Figure 3**: Routers in OSPF backbone (area 0)</a> 

```
                            (External Routes, Redistribute)
                            (Could be towards Internet)
       
                                   172.16.0.0/24 .-------. 172.16.1.0/24
                                      .----------+ R Ext +-------------
                                     /         .2'-------'
                                    /.1
                             .-----+-. 
     . . . . . . . . . . . . | ASBR  | . . . . . . . . . . . . 
   .  AREA 0        .--------+ BB-R3 +--------.                .
 .    (Backbone)    |        '-------'        |                  .
 .                  |        10.0.3.3         |                  .
 .                  |                         |                  .
 .         10.0.1.1 |                         | 10.0.2.2         .
 .              .---+---.                 .---+---.              .
   .            | BB-R1 +-----------------+ BB-R2 |            .
     . . . . . .|  ABR  | . . . . . . . . |  ABR  |. . . . . .
   .  AREA 43   '---+---'                 '---+---'            .
 .                  |                         |                  .
.                   V                         V                   .
.                A43-R1                     A43-R2                .
```


We now turn to the backbone routers, as shown above. *BB-R1* and
*BB-R2* are both area border routers (ABRs). We let their eth0
interface be in area 43, while their *eth1* and *eth2* interfaces are
in area 0. We use BB-R1 to illustrate IP and OSPF configuration.


### IP settings

```
admin@bb-r1:/>
admin@bb-r1:/> configure
admin@bb-r1:/config/> set interface lo ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth0 ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth1 ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth2 ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth0 ipv4 forwarding true
admin@bb-r1:/config/> set interface eth1 ipv4 forwarding true
admin@bb-r1:/config/> set interface eth2 ipv4 forwarding true
admin@bb-r1:/config/> leave
admin@bb-r1:/>
```

> Here we have assumed the use of unnumbered OSPF interfaces, and
> assigned address 10.0.1.1/32 also to *eth0*, which will reside in
> area 43. If the link between BB-R1 and A43-R1 would have used a /30
> subnet, the IP settings would have been adapted to that.

Similar configuration can the be applied to BB-R2 and BB-R3 in [figure
3](#fig3), except that we would leave out interface eth0 on BB-R3.


### OSPF Configuration of Backbone Area

All backbone routers will have their interfaces *eth1* and *eth2* in
area 0. These interfaces are OSPF *point-to-point* interfaces and have
BFD enabled. BB-R1 is used to illustrate. Repeat for BB-R2 and BB-R3.

```
admin@bb-r1:/>
admin@bb-r1:/> configure
admin@bb-r1:/config/> edit routing control-plane-protocol ospfv2 name default
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface lo enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth2 enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 interface-type point-to-point
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth2 interface-type point-to-point
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 bfd enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth2 bfd enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@bb-r1:/>
```

After connecting all backbone routers, we can verify using `show ospf
neighbor` and `show ospf routes`.

```
admin@bb-r1:/>
admin@bb-r1:/> show ospf neighbor

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
10.0.2.2          1 Full/-          18.788s           35.422s 10.0.2.2        eth1:10.0.1.1                        0     0     0
10.0.3.3          1 Full/-          36.632s           33.425s 10.0.3.3        eth2:10.0.1.1                        0     0     0

admin@bb-r1:/>
admin@bb-r1:/> show ospf routes
============ OSPF network routing table ============
N    10.0.1.1/32           [0] area: 0.0.0.0
                           directly attached to lo
N    10.0.2.2/32           [10] area: 0.0.0.0
                           via 10.0.2.2, eth1
N    10.0.3.3/32           [10] area: 0.0.0.0
                           via 10.0.3.3, eth2

============ OSPF router routing table =============

============ OSPF external routing table ===========


admin@bb-r1:/>
```


### Adding external route at R3

To illustrate injection/redistribution of external routes, we let
BB-R3 connect to some external network (172.16.0.0/24) and with a
static route to another network (172.16.1.0/24). BB-R3 will
redistribute these routes into the OSPF domain using OSPF
`redistribute connected` and `redistribute static` settings.
BB-R3 will act as *autonomous system border router* (ASBR).

> In the future, additional routing protocols such as RIP and BGP may
> become supported, and Infix would then be able to redistribute
> routes acquired via those protocols into the OSPF domain.

First we configure IP address to and enable forwarding on *eth3*, and
then a static route to *172.16.1.0/24* (see [figure 3](#fig3)).

``` 
admin@bb-r3:/> configure
admin@bb-r3:/config/> set interface eth3 ipv4 address 172.16.0.1 prefix-length 24
admin@bb-r3:/config/> set interface eth3 ipv4 forwarding true
admin@bb-r3:/config/> edit routing control-plane-protocol static name default
admin@bb-r3:/config/routing/control-plane-protocol/static/name/default/> set ipv4 route 172.16.1.0/24 next-hop next-hop-address 172.16.0.2
admin@bb-r3:/config/routing/control-plane-protocol/static/name/default/> leave
admin@bb-r3:/>
```

Then we configure OSPF to redistribute the connected and static routes
(they will be redistributed as *external type 2* (E2) routes; type is
currently not configurable, but check [Infix documentation][infix-doc]
for updates).

```
admin@bb-r3:/> configure
admin@bb-r3:/config/> edit routing control-plane-protocol ospfv2 name default
admin@bb-r3:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf redistribute connected
admin@bb-r3:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf redistribute static
admin@bb-r3:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@bb-r3:/>
```

At BB-R1 we can now see external routes being advertised by BB-R3.

```
admin@bb-r1:/> show ospf routes
============ OSPF network routing table ============
N    10.0.1.1/32           [0] area: 0.0.0.0
                           directly attached to lo
N    10.0.2.2/32           [10] area: 0.0.0.0
                           via 10.0.2.2, eth1
N    10.0.3.3/32           [10] area: 0.0.0.0
                           via 10.0.3.3, eth2

============ OSPF router routing table =============
R    10.0.3.3              [10] area: 0.0.0.0, ASBR
                           via 10.0.3.3, eth2

============ OSPF external routing table ===========
N E2 172.16.0.0/24         [10/20] tag: 0
                           via 10.0.3.3, eth2
N E2 172.16.1.0/24         [10/20] tag: 0
                           via 10.0.3.3, eth2


admin@bb-r1:/>
```


### ABR Configuration - Connecting Area 43 to the Backbone

Finally, we enable OSPF on interface *eth0* of BB-R1 and BB-R2, and
specify them to be part of area 43. We use BB-R1 as illustration.

```
admin@bb-r1:/> configure
admin@bb-r1:/config/> edit routing control-plane-protocol ospfv2 name default
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth0 enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth0 interface-type point-to-point
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth0 bfd enabled true
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@bb-r1:/>
```

The same configuration is added to BB-R2. After connecting BB-R1
(eth0) to A43-R1 (eth0), and corresponding for BB-R2 to A43-R2 (see
[figure 1](#fig1)), we have connected area 43 to the
backbone. Below we use A43-R1 to inspect the routes exchanged.

```
admin@a43-r1:/> show ospf routes
============ OSPF network routing table ============
N IA 10.0.1.1/32           [10] area: 0.0.0.43
                           via 10.0.1.1, eth0
N IA 10.0.2.2/32           [20] area: 0.0.0.43
                           via 10.0.1.1, eth0
                           via 10.43.2.2, eth1
N IA 10.0.3.3/32           [20] area: 0.0.0.43
                           via 10.0.1.1, eth0
N    10.43.1.0/24          [10] area: 0.0.0.43
                           directly attached to eth3
N    10.43.1.1/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.2.0/24          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.2.2/32          [10] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.3.0/24          [20] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.3.3/32          [10] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.4.0/24          [30] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2
N    10.43.4.4/32          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2

============ OSPF router routing table =============
R    10.0.1.1              [10] area: 0.0.0.43, ABR
                           via 10.0.1.1, eth0
R    10.0.2.2              [20] area: 0.0.0.43, ABR
                           via 10.43.2.2, eth1
R    10.0.3.3           IA [20] area: 0.0.0.43, ASBR
                           via 10.0.1.1, eth0

============ OSPF external routing table ===========
N E2 172.16.0.0/24         [20/20] tag: 0
                           via 10.0.1.1, eth0
N E2 172.16.1.0/24         [20/20] tag: 0
                           via 10.0.1.1, eth0
```

The routing table at A43-R1 can be shown using `show routes`.

```
admin@a43-r1:/> show routes
PREFIX                        NEXT-HOP                          PREF  PROTOCOL
10.0.1.1/32                   10.0.1.1                            20  ospf
10.0.2.2/32                   10.43.2.2                           20  ospf
                              10.0.1.1
10.0.3.3/32                   10.0.1.1                            20  ospf
10.43.1.0/24                  eth3                                 0  kernel
10.43.2.0/24                  10.43.2.2                           20  ospf
10.43.2.2/32                  10.43.2.2                           20  ospf
10.43.3.0/24                  10.43.3.3                           20  ospf
10.43.3.3/32                  10.43.3.3                           20  ospf
10.43.4.0/24                  10.43.2.2                           20  ospf
                              10.43.3.3
10.43.4.4/32                  10.43.2.2                           20  ospf
                              10.43.3.3
172.16.0.0/24                 10.0.1.1                            20  ospf
172.16.1.0/24                 10.0.1.1                            20  ospf
admin@a43-r1:/>
```

Similarly, we can look at the routing table and exchanged routes from
one of the backbone routers (here BB-R1):

```
admin@bb-r1:/> show routes
PREFIX                        NEXT-HOP                          PREF  PROTOCOL
10.0.1.0/24                   eth3                                 0  kernel
10.0.2.2/32                   10.0.2.2                            20  ospf
10.0.3.3/32                   10.0.3.3                            20  ospf
10.43.1.0/24                  10.43.1.1                           20  ospf
10.43.1.1/32                  10.43.1.1                           20  ospf
10.43.2.0/24                  10.43.1.1                           20  ospf
10.43.2.2/32                  10.43.1.1                           20  ospf
10.43.3.0/24                  10.43.1.1                           20  ospf
10.43.3.3/32                  10.43.1.1                           20  ospf
10.43.4.0/24                  10.43.1.1                           20  ospf
10.43.4.4/32                  10.43.1.1                           20  ospf
172.16.0.0/24                 10.0.3.3                            20  ospf
172.16.1.0/24                 10.0.3.3                            20  ospf
admin@bb-r1:/> show ospf routes
============ OSPF network routing table ============
N    10.0.1.1/32           [0] area: 0.0.0.0
                           directly attached to lo
N    10.0.2.2/32           [10] area: 0.0.0.0
                           via 10.0.2.2, eth1
N    10.0.3.3/32           [10] area: 0.0.0.0
                           via 10.0.3.3, eth2
N    10.43.1.0/24          [20] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.1.1/32          [10] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.2.0/24          [30] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.2.2/32          [20] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.3.0/24          [30] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.3.3/32          [20] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.4.0/24          [40] area: 0.0.0.43
                           via 10.43.1.1, eth0
N    10.43.4.4/32          [30] area: 0.0.0.43
                           via 10.43.1.1, eth0

============ OSPF router routing table =============
R    10.0.2.2              [30] area: 0.0.0.43, ABR
                           via 10.43.1.1, eth0
                           [10] area: 0.0.0.0, ABR
                           via 10.0.2.2, eth1
R    10.0.3.3              [10] area: 0.0.0.0, ASBR
                           via 10.0.3.3, eth2

============ OSPF external routing table ===========
N E2 172.16.0.0/24         [10/20] tag: 0
                           via 10.0.3.3, eth2
N E2 172.16.1.0/24         [10/20] tag: 0
                           via 10.0.3.3, eth2


admin@bb-r1:/>
```

With a PC connected to a host subnet, e.g., on 10.43.3.0/24 with
default gw to 10.32.3.1, we could let it ping the external router on
172.16.0.2 (see [figure 1](#fig1)). This assumes the external router
knows it should route the return traffic to 10.43.x.x (and 10.0.x.x)
towards BB-R3 (configuration not shown here).

```
PC3> ping 172.16.0.2

84 bytes from 172.16.0.2 icmp_seq=1 ttl=60 time=1.656 ms
84 bytes from 172.16.0.2 icmp_seq=2 ttl=60 time=1.874 ms
84 bytes from 172.16.0.2 icmp_seq=3 ttl=60 time=1.518 ms
84 bytes from 172.16.0.2 icmp_seq=4 ttl=60 time=1.663 ms
84 bytes from 172.16.0.2 icmp_seq=5 ttl=60 time=1.876 ms

PC3>
```

## Reducing routing state

In the previous section, we could see that a router in a sub-area
(A43-R1) would not only see routes in its own area, but also in the
backbone area as well as routes external to the OSPF
domain. Furthermore, *as there is typically more sub-areas attached to
the backbone, A43-R1 would learn those route prefixes of them too.*
This can be a lot of routes.

In this section we look at two ways to reduce the routing information
that routers in sub-areas, with area 43 as example. The reason could
either be that routers have limited capacity or for administrative
purposes - the organisation running the backbone area may not wish to
expose all routing information to organisation operating the
sub-area(s).

> The drawback with reducing route information is a risk for
> non-optimal routing.

### Avoid distributing external routes from backbone to sub-area.

Assume BB-R3 learns a lot of external routes, e.g., using BGP. A
router in area 43 would not need now about all those routing prefixes;
a default route towards the ABR (BB-R1 or BB-R2) would suffice.
This can be achieved by defining area 43 as an OSPF stub area or NSSA
area. In this example we use NSSA. 

> Please see this [YouTube Clip][youtube-ospf-areas] to learn more on
> differences between stub and NSSA, and when to use which. In the
> example used here, either stub or NSSA could have been used.


On *all* routers in area 43 (internal as well as ABRs), configure area
43 to be an OSPF NSSA area. Here we show how to do it for the A43-R1.

```
admin@a43-r1:/> configure
admin@a43-r1:/config/> edit routing control-plane-protocol ospfv2 name default
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 area-type nssa-area
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@a43-r1:/>
```
Repeat this on BB-R1, BB-R2, A43-R2, A43-R3 and A43-R4.

Then we can watch the routes on a router inside area 43. You will
notice that prefixes for external routes are gone. Instead, the ABRs
inject a default route (0.0.0.0/0) into area 43.

```
admin@a43-r1:/> show ospf routes 
============ OSPF network routing table ============
N IA 0.0.0.0/0             [11] area: 0.0.0.43
                           via 10.0.1.1, eth0
N IA 10.0.1.1/32           [10] area: 0.0.0.43
                           via 10.0.1.1, eth0
N IA 10.0.2.2/32           [20] area: 0.0.0.43
                           via 10.0.1.1, eth0
                           via 10.43.2.2, eth1
N IA 10.0.3.3/32           [20] area: 0.0.0.43
                           via 10.0.1.1, eth0
N    10.43.1.0/24          [10] area: 0.0.0.43
                           directly attached to eth3
N    10.43.1.1/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.2.0/24          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.2.2/32          [10] area: 0.0.0.43
                           via 10.43.2.2, eth1
N    10.43.3.0/24          [20] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.3.3/32          [10] area: 0.0.0.43
                           via 10.43.3.3, eth2
N    10.43.4.0/24          [30] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2
N    10.43.4.4/32          [20] area: 0.0.0.43
                           via 10.43.2.2, eth1
                           via 10.43.3.3, eth2

============ OSPF router routing table =============
R    10.0.1.1              [10] area: 0.0.0.43, ABR
                           via 10.0.1.1, eth0
R    10.0.2.2              [20] area: 0.0.0.43, ABR, ASBR
                           via 10.43.2.2, eth1

============ OSPF external routing table ===========


admin@a43-r1:/> show routes 
PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
0.0.0.0/0                     10.0.1.1                            20  ospf      
10.0.1.1/32                   10.0.1.1                            20  ospf      
10.0.2.2/32                   10.0.1.1                            20  ospf      
                              10.43.2.2                     
10.0.3.3/32                   10.0.1.1                            20  ospf      
10.43.1.0/24                  eth3                                 0  kernel    
10.43.2.0/24                  10.43.2.2                           20  ospf      
10.43.2.2/32                  10.43.2.2                           20  ospf      
10.43.3.0/24                  10.43.3.3                           20  ospf      
10.43.3.3/32                  10.43.3.3                           20  ospf      
10.43.4.0/24                  10.43.2.2                           20  ospf      
                              10.43.3.3                     
10.43.4.4/32                  10.43.2.2                           20  ospf      
                              10.43.3.3                     
admin@a43-r1:/> 
```

This may seem like a small reduction, but we only redistribute a few
external routes into the OSPF domain. With more external routes (e.g.,
if BGP routes are distributed), the reduction would be more apparent.

> Only sub-areas can be configured area type *stub* or *NSSA*. Routers
> in the backbone would see all external routes.


### Avoid distributing route prefixes (summary routes) into a sub-area

As the ABRs injects a default route into a stub or NSSA area, routers
inside area 43 do not need to know about all route prefixes in the
backbone, *or in other sub-areas*.

By using OSPF setting `summary false` in the ABRs, they will skip
distributing prefixes. (Route prefixes are referred to as *summary
routes* in OSPF for IPv4, which is different from *route
summarization*; a topic for a future blog post.) Below we show it for
BB-R1.

```
admin@bb-r1:/> configure
admin@bb-r1:/config/> edit routing control-plane-protocol ospfv2 name default
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 summary false
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@bb-r1:/> copy running-config startup-config
admin@bb-r1:/>
```
After repeating this on BB-R2, we can inspect the affect on the
routers inside area 43. All prefixes outside area 43 are hidden from
these routers. 

```
admin@a43-r1:/> show routes
PREFIX                        NEXT-HOP                          PREF  PROTOCOL
0.0.0.0/0                     10.0.1.1                            20  ospf
10.43.1.0/24                  eth3                                 0  kernel
10.43.2.0/24                  10.43.2.2                           20  ospf
10.43.2.2/32                  10.43.2.2                           20  ospf
10.43.3.0/24                  10.43.3.3                           20  ospf
10.43.3.3/32                  10.43.3.3                           20  ospf
10.43.4.0/24                  10.43.3.3                           20  ospf
                              10.43.2.2
10.43.4.4/32                  10.43.3.3                           20  ospf
                              10.43.2.2
admin@a43-r1:/>
```

> Here we skip forwarding prefix information about destinations within
> the backbone area. With multiple sub-areas connected to the
> backbone, the reduction of routes and routing state to keep within
> sub-areas would be even more obvious.

The drawback may be non-optimal routing, as without further
information the traffic may exit the area through one ABR while some
other may much closer to the destination. Thus, if routers can cope
with the route information, *regular* area type may be the preferred
choice for sub-areas, as opposed to types NSSA and stub.

[infix-unnumbered-post]: ../../../../2024/01/30/using-ospf-unnumbered-interfaces-with-infix-linux-revisited
[infix-doc]: https://github.com/kernelkit/infix/tree/main/doc
[youtube-ospf-areas]: https://www.youtube.com/watch?v=cM3OI_ZyRuQ
