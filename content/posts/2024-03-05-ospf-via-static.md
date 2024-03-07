---
title:  "Connecting OSPF domains via static route(s)"
author: Jon-Olov Vatn
date:   2024-03-05 17:07:42 +0200
tags:
 - linux
 - networking
 - infix
 - ospf
 - static route
 - originate default route
 - default route advertise
---

In an earlier [blog post][infix-area-post] we connected two OSPF areas
to form a larger OSPF routing domain. (Another way to look at it is
that we divided the larger domain into more manageable areas. Still,
it was one OSPF domain.)  In this post, we again look at two areas,
but chose to connect them via a static route, or possibly two static
routes for redundancy.  The motivation for using this approach is if
the OSPF domains are run by two separate organizations, and policy
reasons hinder them to build a common large OSPF domain. Each
organisation will manage their own routing domain using a dynamic
routing protocol (here OSPF), but they will be interconnected by a
static route, or two.

It is recommended to first look at the previous [blog
post][infix-area-post] on OSPF areas in Infix before reading this one.

## Overview of intended topology

The topology in this blog post is intended to be hierarchical. There is
a main organisation with a *backbone*, to which one ore more
sub-organizations can be connected. The difference as compared to the
previous [blog post][infix-area-post] is that each organisation runs
its own OSPF routing domain. It is even possible for different
organizations to run different dynamic routing protocols internally,
but here we use OSPF as that is what Infix currently supports.

As domains are separate, each organisation can use area 0 (backbone),
but we are reusing the area assignments and (to a large extent) the IP
plans from the the previous [blog post][infix-area-post].


<a name="fig1">**Figure 1**: Overview of Example Topology</a> 

```

                             .-------. 
     . . . . . . . . . . . . |       | . . . . . . . . . . . . 
   .  AREA 0        .--------+ BB-R3 +--------.                 .
 .    (Backbone)    |        '-------'        |                  .
 .    Main Org.     |        10.0.3.3         |                  .
 .                  |                         |                  .
 .         10.0.1.1 |                         | 10.0.2.2         .
 .              .---+---.                 .---+---.              .
   .            | BB-R1 +-----------------+ BB-R2 |            .
     . . . . . .| ASBR  | . . . . . . . . | ASBR  |. . . . . .
                '---+---'                 '---+---'
                  .1|                       .5.
                    |                         .  Possible
        10.0.1.0/30 |             10.0.1.4/30 .  Redundant
                    |                         .  Uplink
                  .2|                       .6.
  . . . . . . . .---+----. . . . . . . . .----+---. . . . . . . .
 . AREA 43      | ASBR   |               | (ASBR) |  IP Range     . 
.  Org. X       | A43-R1 +---------------+ A43-R2 |  10.43.0.0/16  .
.    .-.        '-+-+----'               '----+-+-'        .-.     .
.    |H|         /  |10.43.1.1       10.43.2.2|  \         |H|     .
.    '+'        /   |                         |   \        '+'     .
.     |     .1 /    |                         |    \ .1     |      .
.  ---+-------´     |                         |     `-------+--    .
.   10.43.1.0/24    |                         |   10.43.2.0/24     .
.                   |10.43.3.3       10.43.4.4|                    .
.    .-.        .---+----.               .----+---.        .-.     .
.    |H|        | A43-R3 +---------------+ A43-R4 |        |H|     .
.    '+'        '-+------'               '------+-'        '+'     .
.     |       .1 /                               \ .1       |      .
.  ---+---------´                                 `---------+--    .
.   10.43.3.0/24                                  10.43.4.0/24     .
  .                                                              .
    . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . . 
```

## The sub-domain

Despite being an OSPF domain of its own, we he refer to bottom network
run by organisation X as a *sub-domain*. The top router A43-R1 will
have a default route point upwards towards BB-R1. It is assumed that
the backbone domain could have multiple sub-domains connected to it
via static routes, although only one sub-domain is shown here.

Getting redundancy to work with static routes is trickier, however, we
will extend the example to let A43-R2 to act as a redundant router
towards BB-R2 at the end of this post.

> To make redundancy work well with static routes, it is preferred if
> uplinks can make use of bidirectional forwarding detection (BFD),
> however, Infix does not yet support that for static links.

<a name="fig2">**Figure 2**: Routers in sub-domain</a> 

```
                  BB-R1                    (BB-R2)
             Uplink ^                         ^  Possible
    (Default Route) |                         .  Redundant
	                |                         .  Uplink			
	    10.0.1.2/30 |            (10.0.1.6/30).  (Default Route)
..............  .---+----.  ...........  .----+---. ................
   AREA 43      | ASBR   |               | (ASBR) |  IP Range
   Org. X       | A43-R1 +---------------+ A43-R2 |  10.43.0.0/16
     .-.        '-+-+----'               '----+-+-'        .-.
     |H|         /  |10.43.1.1       10.43.2.2|  \         |H|
     '+'        /   |                         |   \        '+'
      |     .1 /    |                         |    \ .1     |
   ---+-------´     |                         |     `-------+--
    10.43.1.0/24    |                         |   10.43.2.0/24
                    |10.43.3.3       10.43.4.4|
     .-.        .---+----.               .----+---.        .-.
     |H|        | A43-R3 +---------------+ A43-R4 |        |H|
     '+'        '-+------'               '------+-'        '+'
      |       .1 /                               \ .1       |
   ---+---------´                                 `---------+--
    10.43.3.0/24                                  10.43.4.0/24
```

For OSPF configuration, we have assumed the following:

- OSPF unnumbered interfaces are used for links between routers
- Bidirectional Forwarding Detection (BFD) is used for fast link-down
  detection.

For more information, see previous [blog post][infix-unnumbered-post]
on using unnumbered OSPF interfaces. 

Router 1 (A43-R1) is used as illustration. We start by configuring a
suitable hostname.

```
admin@infix-23-00-00:~$ cli
admin@infix-23-00-00:/> configure 
admin@infix-23-00-00:/config/> set system hostname a43-r1
admin@infix-23-00-00:/config/> leave
admin@a43-r1:/> copy running-config startup-config 
admin@a43-r1:/> 
```

> Remember to `copy running startup` to store your changes
> permanently after leaving configuration context. This will not be
> shown in later examples.

### IP address and forwarding settings on R1

As show in [figure 2](#fig2) R1 (A43-R1) needs four Ethernet
interfaces.  To see what interfaces we have, use `show interfaces`.

```
admin@a43-r1:/> show interfaces 
INTERFACE       PROTOCOL   STATE       DATA
eth0            ethernet   DOWN        0c:16:14:23:00:00
eth1            ethernet   DOWN        0c:16:14:23:00:01
eth2            ethernet   DOWN        0c:16:14:23:00:02
eth3            ethernet   DOWN        0c:16:14:23:00:03
eth4            ethernet   DOWN        0c:16:14:23:00:04
lo              ethernet   UP          00:00:00:00:00:00
                ipv4                   127.0.0.1/8 (static)
                ipv6                   ::1/128 (static)
admin@a43-r1:/> 
```

R1 and the other routers in this example has five Ethernet interfaces
named *eth0*..*eth4*. We use *eth0* as uplink towards BB-R1, *eth1*
and *eth2* for links to other routers in area 43, and *eth3* for the
local subnet (10.43.1.0/24).

```
admin@a43-r1:/> 
admin@a43-r1:/> configure
admin@a43-r1:/config/> set interface lo ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth1 ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth2 ipv4 address 10.43.1.1 prefix-length 32
admin@a43-r1:/config/> set interface eth3 ipv4 address 10.43.1.1 prefix-length 24
admin@a43-r1:/config/> set interface eth0 ipv4 address 10.1.0.2 prefix-length 30
admin@a43-r1:/config/> set interface eth0 ipv4 forwarding 
admin@a43-r1:/config/> set interface eth1 ipv4 forwarding 
admin@a43-r1:/config/> set interface eth2 ipv4 forwarding 
admin@a43-r1:/config/> set interface eth3 ipv4 forwarding 
admin@a43-r1:/config/> 
```

To list the changes made, the `diff` command is used. Note that
boolean settings such `set forwarding` is expanded to `forwarding
true`. (Similarly, `no forwarding` would expand to `forwarding
false`.)

```
admin@a43-r1:/config/> diff
interfaces {
  interface eth0 {
+    ipv4 {
+      forwarding true;
+      address 10.1.0.2 {
+        prefix-length 30;
+      }
+    }
  }
  interface eth1 {
+    ipv4 {
+      forwarding true;
+      address 10.43.1.1 {
+        prefix-length 32;
+      }
+    }
  }
  interface eth2 {
+    ipv4 {
+      forwarding true;
+      address 10.43.1.1 {
+        prefix-length 32;
+      }
+    }
  }
  interface eth3 {
+    ipv4 {
+      forwarding true;
+      address 10.43.1.1 {
+        prefix-length 24;
+      }
+    }
  }
  interface lo {
    ipv4 {
+      address 10.43.1.1 {
+        prefix-length 32;
+      }
    }
  }
}
admin@a43-r1:/config/> leave
admin@a43-r1:/> 
```

Running `show interfaces` again would show the following:

```
admin@a43-r1:/> show interfaces 
INTERFACE       PROTOCOL   STATE       DATA
eth0            ethernet   DOWN        0c:16:14:23:00:00
                ipv4                   10.1.0.2/30 (static)
eth1            ethernet   DOWN        0c:16:14:23:00:01
                ipv4                   10.43.1.1/32 (static)
eth2            ethernet   DOWN        0c:16:14:23:00:02
                ipv4                   10.43.1.1/32 (static)
eth3            ethernet   DOWN        0c:16:14:23:00:03
                ipv4                   10.43.1.1/24 (static)
eth4            ethernet   DOWN        0c:16:14:23:00:04
lo              ethernet   UP          00:00:00:00:00:00
                ipv4                   127.0.0.1/8 (static)
                ipv4                   10.43.1.1/32 (static)
                ipv6                   ::1/128 (other)
admin@a43-r1:/> 
```

The settings for interfaces *eth1*, *eth2* and *lo* can be repeated on
the other routers in the domain (R2, R3, and R4), adapting the
addresses in line with [figure 2](#fig2). On R2, you may also add a
corresponding address on interface *eth0* (10.1.0.6/30).

### OSPF settings on R1

It is time to configure OSPF settings. 

- Enable OSPF on the relevant interfaces
- Mark all *inter-router interfaces* as OSPF point-to-point
  interfaces.  (This combined with /32 IP addresses will make them an
  OSPF unnumbered interfaces.)
- Enable BFD on all inter-router interfaces within the OSPF domain
  (for fast link-down  detection and OSPF route convergence).
- We set eth3 as OSPF passive interface (as there is no router to talk
  to on that LAN)

```
admin@a43-r1:/> configure
admin@a43-r1:/config/> edit routing control-plane-protocol ospfv2 name default 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface lo enabled 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth1 enabled 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth2 enabled 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth3 enabled 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth1 interface-type point-to-point 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth2 interface-type point-to-point 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth1 bfd enabled 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth2 bfd enabled 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.43 interface eth3 passive 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@a43-r1:/> 
```

Now, repeat OSPF settings for routers R2, R3 and R4 in area 43 (see
[figure 2](#fig2)), and connect the routers via their respective eth1 and eth2. As we use
unnumbered interfaces, it does not matter if R1 connects eth1 or eth2
to R2, etc.

When done, we can verify if routers are able to discover each other and
exchange routing information. E.g., on R1 we can run `show ospf
neighbor` to see if R1 can see R2 (10.43.2.2) and R3 (10.43.3.3).

```
admin@a43-r1:/> show ospf neighbor 

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
10.43.2.2         1 Full/-          7m12s             37.087s 10.43.2.2       eth1:10.43.1.1                       0     0     0
10.43.3.3         1 Full/-          2m07s             31.857s 10.43.3.3       eth2:10.43.1.1                       0     0     0

admin@a43-r1:/> 
```

The routes exchanged would look like follows (assuming interfaces on
the host subnets have not yet got link up; otherwise some /24 networks
would be seen too).

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


### Configuring and advertising a default route

Router R1 (A43-R1) in the sub-domain is the gateway towards the
backbone network, see [figure 1](#fig2). As we assume that the
backbone in turn can be connected to the Internet, as well as to other
sub-domains, we let R1 have a *default route* (0.0.0.0/0) set towards
BB-R1.

To inform other routers in the sub-domain about the default route, R1
will advertise it via OSPF.

First we add the static default route.

```
admin@a43-r1:/> configure 
admin@a43-r1:/config/> edit routing control-plane-protocol static name default 
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> set ipv4 route 0.0.0.0/0 next-hop next-hop-address 10.1.0.1
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> leave
admin@a43-r1:/> 
```
Then we let R1 advertise reachability to the default route via  OSPF. 
  
```
admin@a43-r1:/> configure
admin@a43-r1:/config/> edit routing control-plane-protocol ospfv2 name default 
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf default-route-advertise enabled
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> leave
admin@a43-r1:/>
```

Here we configured R1 to only advertise the default route in OSPF if
R1 itself has an active default route. (That is, we do **not** set the
advertise `always` primitive.)  Thus, if the link towards BB-R1 is
up, the default route would be seen by other OSPF routers as shown
for R3 below.

```
admin@a43-r3:/> show ospf routes 
============ OSPF network routing table ============
N    10.43.1.1/32          [10] area: 0.0.0.43
                           via 10.43.1.1, eth1
N    10.43.2.2/32          [20] area: 0.0.0.43
                           via 10.43.1.1, eth1
                           via 10.43.4.4, eth2
N    10.43.3.3/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.4.4/32          [10] area: 0.0.0.43
                           via 10.43.4.4, eth2

============ OSPF router routing table =============
R    10.43.1.1             [10] area: 0.0.0.43, ASBR
                           via 10.43.1.1, eth1

============ OSPF external routing table ===========
N E2 0.0.0.0/0             [10/10] tag: 0
                           via 10.43.1.1, eth1


admin@a43-r3:/> 
```


## Backbone routers


<a name="fig3">**Figure 3**: Routers in OSPF backbone (area 0)</a> 

```
                             .-------. 
     . . . . . . . . . . . . |       | . . . . . . . . . . . . 
   .  AREA 0        .--------+ BB-R3 +--------.                .
 .    (Backbone)    |        '-------'        |                  .
 .                  |        10.0.3.3         |                  .
 .                  |                         |                  .
 .         10.0.1.1 |                         | 10.0.2.2         .
 .              .---+---.                 .---+---.              .
   .            | BB-R1 +-----------------+ BB-R2 |            .
     . . . . . .| ASBR  | . . . . . . . . | ASBR  |. . . . . .
                '---+---'                 '---+---'
        10.0.1.1/30 |             10.0.1.5/30 . Possible 
                    |                         . Redundant
       Route to     |          (Route to      . Link    					
       10.43.0.0/16 V           10.43.0.0/16) V
                 A43-R1                   (A43-R2)
```


We now turn to the backbone routers. These routers also use dynamic
routing to exchange routing information, here illustrated by OSPF.  To
start with, only BB-R1 has a connection to the sub-domain with a
connection to A43-R1, but we can prepare configuration for a redundant
link between BB-R2 and A43-R2.

### IP settings

IP settings for BB-R1 is shown below. We let its *eth0* interface connect to A43-R1
using static routing, while *eth1* and *eth2* interfaces connect to
the OSPF routers. Unnumbered OSPF interfaces are assumed, thus the
same /32 address is assigned to *lo*, *eth1* and *eth2*.

```
admin@bb-r1:/> configure 
admin@bb-r1:/config/> set interface eth0 ipv4 address 10.1.0.1 prefix-length 30
admin@bb-r1:/config/> set interface lo ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth1 ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth2 ipv4 address 10.0.1.1 prefix-length 32
admin@bb-r1:/config/> set interface eth0 ipv4 forwarding 
admin@bb-r1:/config/> set interface eth1 ipv4 forwarding 
admin@bb-r1:/config/> set interface eth2 ipv4 forwarding 
admin@bb-r1:/config/> leave
admin@bb-r1:/> 
```

Similar configuration can the be applied to BB-R2 and BB-R3 in [figure
3](#fig3), except that we would leave out interface eth0 on
BB-R3. (BB-R2 can have address 10.1.0.5/30 assigned to its *eth0*.)


### OSPF Configuration of Backbone Area

All backbone routers will have their interfaces *eth1* and *eth2* in
area 0. These interfaces are OSPF *point-to-point* interfaces and have
BFD enabled. BB-R1 is used to illustrate. Repeat for BB-R2 and BB-R3.

```
admin@bb-r1:/> configure 
admin@bb-r1:/config/> edit routing control-plane-protocol ospfv2 name default 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface lo enabled 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 enabled 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth2 enabled 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 interface-type point-to-point 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth2 interface-type point-to-point 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 bfd enabled  
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth2 bfd enabled  
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@bb-r1:/> 
```

After connecting all backbone routers, we can verify using `show ospf
neighbor` and `show ospf routes`.

```
admin@bb-r1:/> show ospf neighbor 

Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
10.0.2.2          1 Full/-          1m24s             35.213s 10.0.2.2        eth1:10.0.1.1                        0     0     0
10.0.3.3          1 Full/-          24.920s           39.269s 10.0.3.3        eth2:10.0.1.1                        0     0     0

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

### Static route towards sub-domain

<a name="fig4">**Figure 4**: Connecting the two OSPF domains</a> 
```
      [BB-R3]
      /     \      OSPF Domain
     /       \     "Backbone"
[BB-R1]------[BB-R2]
    |
	| Static
	| Route
	|
[A43-R1]----[A43-R2]	
    |           | OSPF Domain
    |           | "Sub-domain"
[A43-R3]----[A43-R4]	
```

The sub-domain can already reach the "backbone", as A43 has a default
route set towards BB-R1. In order for the Backbone domain to reach the
sub-domain, we let BB-R1 have a static route for prefix 10.43.0.0/16
towards A43-R1.

```
admin@bb-r1:/> configure 
admin@bb-r1:/config/> edit routing control-plane-protocol static name default 
admin@bb-r1:/config/routing/control-plane-protocol/static/name/default/> set ipv4 route 10.43.0.0/16 next-hop next-hop-address 10.1.0.2
admin@bb-r1:/config/routing/control-plane-protocol/static/name/default/> leave
admin@bb-r1:/> 
```

and then we let BB-R1 redistribute this route into the backbone OSPF
domain.

```
admin@bb-r1:/> configure 
admin@bb-r1:/config/> edit routing control-plane-protocol ospfv2 name default 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf redistribute static 
admin@bb-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@bb-r1:/> 
```

If we then connect the domains we can check routing information and
connectivity from BB-R3 by pinging A43-R3 (10.43.3.3).

```
admin@bb-r3:/> show ospf routes 
============ OSPF network routing table ============
N    10.0.1.1/32           [10] area: 0.0.0.0
                           via 10.0.1.1, eth2
N    10.0.2.2/32           [10] area: 0.0.0.0
                           via 10.0.2.2, eth1
N    10.0.3.3/32           [0] area: 0.0.0.0
                           directly attached to lo

============ OSPF router routing table =============
R    10.0.1.1              [10] area: 0.0.0.0, ASBR
                           via 10.0.1.1, eth2

============ OSPF external routing table ===========
N E2 10.43.0.0/16          [10/20] tag: 0
                           via 10.0.1.1, eth2


admin@bb-r3:/> show routes 
PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
10.0.1.1/32                   10.0.1.1                            20  ospf      
10.0.2.2/32                   10.0.2.2                            20  ospf      
10.43.0.0/16                  10.0.1.1                            20  ospf      
admin@bb-r3:/> ping 10.43.3.3
PING 10.43.3.3 (10.43.3.3) 56(84) bytes of data.
64 bytes from 10.43.3.3: icmp_seq=1 ttl=62 time=1.64 ms
64 bytes from 10.43.3.3: icmp_seq=2 ttl=62 time=2.74 ms
^C
--- 10.43.3.3 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 1.639/2.188/2.737/0.549 ms
admin@bb-r3:/> 
```

## Use black-hole route to avoid routing loops 


<a name="fig5">**Figure 5**: Issue with routing loops</a> 
```
      [BB-R3]
      /     \      OSPF Domain
     /       \     "Backbone"
[BB-R1]------[BB-R2]
    | |
	| V 10.43.0.0/16   ^
	|                  | Dst:10.43.100.5 
	| ^ 0.0.0.0/0      V
	| |
[A43-R1]----[A43-R2]	
    |           | OSPF Domain
    |           | "Sub-domain"
[A43-R3]----[A43-R4]	
```
[Figure 5](#fig5) illustrates an issue that can occur when static
routes include an aggregated prefix. Lets see what happens if BB-R3
would send a ping to 10.43.100.5. The packet would reach BB-R1 who
forwards it to A43-R1 as it matches 10.43.0.0/16. But there is no
network or route within the sub-domain that matches
10.43.100.5. Therefore A43-R1 sends it to its default route, i.e.,
back to BB-R1. And these routers will forward the packet back and
forth until TTL reaches 0 and the packet is dropped.

To avoid this, we let A43-R1 set up a *black-hole* route to
10.43.0.0/16, meaning it will drop any packet in this range
immediately unless it has a more specific route to the destination. 

```
admin@a43-r1:/> configure 
admin@a43-r1:/config/> edit routing control-plane-protocol static name default 
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> set ipv4 route 10.43.0.0/16 next-hop special-next-hop blackhole 
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> leave
admin@a43-r1:/> 
```

## Adding redundancy between the domains


<a name="fig6">**Figure 6**: Redundancy between the OSPF domains</a> 
```
      [BB-R3]
      /     \      OSPF Domain
     /       \     "Backbone"
[BB-R1]------[BB-R2]
    |           |
	|     ^     |
	|     |     |
	|  Static   |
	|  Routes   |
	|     |     |
	|     V     |
	|           |
[A43-R1]----[A43-R2]	
    |           | OSPF Domain
    |           | "Sub-domain"
[A43-R3]----[A43-R4]	
```
The objective of adding a second connection between the two OSPF
domains is to enable redundancy in case BB-R1 or A43-R1 or the link
between the goes down. But providing route convergence in case of
failing links or other topology changes is the task of routing
protocols such as OSPF. Achieving fail-over with static routing is
tricky, and only works under certain circumstances.

In this example achieve redundancy and fail-over, given the following
assumptions:

1. The routers on either side of the static route (A43-R1 and BB-R1,
   respective A43-R2 and BB-R2) will get a physical link-down if the
   static link between the goes down. (Even better would be if
   *bidirectional forwarding detection* (BFD) was enabled, as that
   would detect logical link-down too. However, *Infix* currently only
   support BFD for OSPF links.)
1. Multiple failures need not be handled. E.g., if BB-R1 loses its
   links towards BB-R2 and BB-R3, A43-R1 will continue to forward
   packets to the backbone domain towards BB-R1.
1. Asymmetric routing is acceptable. Symmetric routing can be achieved
   if one of the router pairs (say A43-R2 and BB-R2) would inject
   their static routes with higher cost than the other pair. Infix
   currently does not support configuring cost of redistributed
   routes.
   
   
To achieve redundancy, BB-R2 should mimic BB-R1 in terms of IP
setting and routes for the static link (interface *eth0* in the
examples shown), as well as redistributing the route into OSPF.
The same goes for A43-R2 mimicking A43-R1.

In the example below, we let A43-R3 ping BB-R1. The first 8 packets go
the shortest path. Then we break the link between A43-R1 and
BB-R1. The packets will then take the somewhat longer path via A43-R2
and BB-R2.

```
admin@a43-r3:/> ping 10.0.1.1
PING 10.0.1.1 (10.0.1.1) 56(84) bytes of data.
64 bytes from 10.0.1.1: icmp_seq=1 ttl=63 time=0.962 ms
64 bytes from 10.0.1.1: icmp_seq=2 ttl=63 time=1.44 ms
64 bytes from 10.0.1.1: icmp_seq=3 ttl=63 time=1.49 ms
64 bytes from 10.0.1.1: icmp_seq=4 ttl=63 time=1.73 ms
64 bytes from 10.0.1.1: icmp_seq=5 ttl=63 time=0.894 ms
64 bytes from 10.0.1.1: icmp_seq=6 ttl=63 time=1.44 ms
64 bytes from 10.0.1.1: icmp_seq=7 ttl=63 time=1.95 ms
64 bytes from 10.0.1.1: icmp_seq=8 ttl=63 time=1.43 ms
64 bytes from 10.0.1.1: icmp_seq=10 ttl=61 time=2.21 ms   <=== New route
64 bytes from 10.0.1.1: icmp_seq=11 ttl=61 time=2.65 ms   (TTL changed)
64 bytes from 10.0.1.1: icmp_seq=12 ttl=61 time=2.09 ms
64 bytes from 10.0.1.1: icmp_seq=13 ttl=61 time=1.61 ms
64 bytes from 10.0.1.1: icmp_seq=14 ttl=61 time=2.51 ms
64 bytes from 10.0.1.1: icmp_seq=15 ttl=61 time=3.41 ms
64 bytes from 10.0.1.1: icmp_seq=16 ttl=61 time=3.64 ms

--- 10.0.1.1 ping statistics ---
16 packets transmitted, 15 received, 6.25% packet loss, time 15083ms
rtt min/avg/max/mdev = 0.894/1.963/3.639/0.779 ms
admin@a43-r3:/> 
```
   
[infix-unnumbered-post]: ../../../../2024/01/30/using-ospf-unnumbered-interfaces-with-infix-linux-revisited
[infix-area-post]: ../../../../2024/02/07/using-ospf-areas-in-infix
[infix-doc]: https://github.com/kernelkit/infix/tree/main/doc



