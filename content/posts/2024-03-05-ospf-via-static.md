---
title:  "Connecting OSFP domains via static route(s)"
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

In an earlier [blog post][infix-area-post] we connected two OSPF
areas to form a larger OSPF routing domain. (Another way to look at it
is that we devided the larger domain into more manageable areas.)
In this post, we again look at two areas, but chose to connect them
via a static route, or possibly two static routes for redundancy.
The motivation for using this approach is if the OSPF domains are run
by two separate organisations, and policy reasons hinder them to build
a common OSPF domain.

It is recommended to first look at the previous [blog
post][infix-area-post] on OSPF areas in Infix before reading this one.

## Overview of intended topology

The topology in this blog is intended to be hiearchical. There is a
main organisation with a *backbone*, to which one ore more
sub-organisations can be connected. The difference as compared to  the previous [blog
post][infix-area-post] is that each organisation runs its own OSPF
routing domain. It is even possible for different organisations to run
different dynamic routing protocols internally, but here we use OSPF
as that is what Infix currently supports.

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
have a default route point upwards towards BB-R1. 

Getting redundancy to work with static routes is tricker, however, we
will extend the example to let A43-R2 to act as a redundant router
towards BB-R2.

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

R1 and the other routers in this example has six Ethernet interfaces
named *eth0*..*eth5*. We use *eth0* as uplink towards BB-R1, *eth1*
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

### Static routes on R1

Router A43-R1 in the sub-domain needs two static route settings:

- one *default route* pointing to BB-R1. This is how traffic from the
  sub-domain will reach the outside world (here limited to the
  *backbone* run by the main organisation)
- one *blackhole route* for prefix 10.43.0.0/16 to avoid that unknown
  routes within that prefix, e.g., packets for 10.43.63.34 is sent
  along the default route to BB-R1 (which would then send it back to
  A43-R1, causing a routing loop).

```
admin@a43-r1:/> configure 
admin@a43-r1:/config/> edit routing control-plane-protocol static name default 
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> set ipv4 route 0.0.0.0/0 next-hop next-hop-address 10.1.0.1
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> set ipv4 route 10.43.0.0/16 next-hop special-next-hop blackhole 
admin@a43-r1:/config/routing/control-plane-protocol/static/name/default/> leave
admin@a43-r1:/> 
```
In case R2 (A43-R2) should act as a redundant uplink router, similar
configurations will be added there too, be it with the next-hop of default
route pointing to BB-R2 (10.1.0.5). We suggest skipping this for now,
and instead return to this in a later section.

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
- We let R1 advertise reachability to the default route (0.0.0.0/0),
  however, R1 only does this when it itself has an active default
  route. (That is, we do **not** set the advertise `always`
  primitive.) 

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
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf default-route-advertise enable
admin@a43-r1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
admin@a43-r1:/> 
```

Now, repeat OSPF settings for routers R2, R3 and R4 in area 43 (see
[figure 2](#fig2)), **except** that we **leave out** `set ospf
default-route-advertise`. (Later on we return to R2 and add it there
too.)

Then connect the routers via their respective eth1 and eth2. As we use
unnumbered interfaces, it does not matter if R1 connects eth1 to R2
and eth2 to R3 or vice versa.

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

### Advertising default route

Router R1 in [figure 2](#fig2)) has been configured to advertise a
default route into the OSPF domain, however, only if it has an active
default route itself. As the uplink interface (*eth0*) is down, R1
lacks an active default route.

```
admin@a43-r1:/> show routes 
PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
10.1.0.0/30                   eth0                                 0  kernel    
10.43.0.0/16                  blackhole                           20  static    
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

If we look at another router such as R3, it has not received any route
to 0.0.0.0/0 via OSPF.

```
admin@a43-r3:/> show ospf routes 
============ OSPF network routing table ============
N    10.43.1.0/24          [20] area: 0.0.0.43
                           via 10.43.1.1, eth1
N    10.43.1.1/32          [10] area: 0.0.0.43
                           via 10.43.1.1, eth1
N    10.43.2.0/24          [30] area: 0.0.0.43
                           via 10.43.1.1, eth1
                           via 10.43.4.4, eth2
N    10.43.2.2/32          [20] area: 0.0.0.43
                           via 10.43.1.1, eth1
                           via 10.43.4.4, eth2
N    10.43.3.0/24          [10] area: 0.0.0.43
                           directly attached to eth3
N    10.43.3.3/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.4.0/24          [20] area: 0.0.0.43
                           via 10.43.4.4, eth2
N    10.43.4.4/32          [10] area: 0.0.0.43
                           via 10.43.4.4, eth2

============ OSPF router routing table =============
R    10.43.1.1             [10] area: 0.0.0.43, ASBR
                           via 10.43.1.1, eth1

============ OSPF external routing table ===========


admin@a43-r3:/> 
admin@a43-r3:/> show routes
PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
10.43.1.0/24                  10.43.1.1                           20  ospf      
10.43.1.1/32                  10.43.1.1                           20  ospf      
10.43.2.0/24                  10.43.1.1                           20  ospf      
                              10.43.4.4                     
10.43.2.2/32                  10.43.1.1                           20  ospf      
                              10.43.4.4                     
10.43.3.0/24                  eth3                                 0  kernel    
10.43.4.0/24                  10.43.4.4                           20  ospf      
10.43.4.4/32                  10.43.4.4                           20  ospf      
admin@a43-r3:/> 
```

If we then bring up interface *eth0* on R1, by connecting it to BB-R1,
R1's configured default route will become active ...

```
admin@a43-r1:/> show routes 
PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
0.0.0.0/0                     10.1.0.1                            20  static    
10.1.0.0/30                   eth0                                 0  kernel    
10.43.0.0/16                  blackhole                           20  static    
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

... and R3 and other routers in the OSPF domain will receive a default
route too.

```
admin@a43-r3:/> show ospf routes 
============ OSPF network routing table ============
N    10.43.1.0/24          [20] area: 0.0.0.43
                           via 10.43.1.1, eth1
N    10.43.1.1/32          [10] area: 0.0.0.43
                           via 10.43.1.1, eth1
N    10.43.2.0/24          [30] area: 0.0.0.43
                           via 10.43.1.1, eth1
                           via 10.43.4.4, eth2
N    10.43.2.2/32          [20] area: 0.0.0.43
                           via 10.43.1.1, eth1
                           via 10.43.4.4, eth2
N    10.43.3.0/24          [10] area: 0.0.0.43
                           directly attached to eth3
N    10.43.3.3/32          [0] area: 0.0.0.43
                           directly attached to lo
N    10.43.4.0/24          [20] area: 0.0.0.43
                           via 10.43.4.4, eth2
N    10.43.4.4/32          [10] area: 0.0.0.43
                           via 10.43.4.4, eth2

============ OSPF router routing table =============
R    10.43.1.1             [10] area: 0.0.0.43, ASBR
                           via 10.43.1.1, eth1

============ OSPF external routing table ===========
N E2 0.0.0.0/0             [10/10] tag: 0
                           via 10.43.1.1, eth1


admin@a43-r3:/> show routes
PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
0.0.0.0/0                     10.43.1.1                           20  ospf      
10.43.1.0/24                  10.43.1.1                           20  ospf      
10.43.1.1/32                  10.43.1.1                           20  ospf      
10.43.2.0/24                  10.43.1.1                           20  ospf      
                              10.43.4.4                     
10.43.2.2/32                  10.43.1.1                           20  ospf      
                              10.43.4.4                     
10.43.3.0/24                  eth3                                 0  kernel    
10.43.4.0/24                  10.43.4.4                           20  ospf      
10.43.4.4/32                  10.43.4.4                           20  ospf      
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
[infix-area-post]: ../../../../2024/02/07/using-ospf-areas-in-infix
[infix-doc]: https://github.com/kernelkit/infix/tree/main/doc
[youtube-ospf-areas]: https://www.youtube.com/watch?v=cM3OI_ZyRuQ
