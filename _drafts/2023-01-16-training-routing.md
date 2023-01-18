---
title:  Network Training — Routing
author: Joachim Wiberg
date:   2023-01-16 09:58:42 +0100
---

Lab exercises suitable for practicing bridging, VLANs, and routing
between devices on different subnets.

<!-- more -->

## Routing between VLANs

Extend the previous exercise with a router.  The figure below depicts
the classic *"router on a stick"* setup where the router is attached
to the switch(es) using a VLAN trunk.

          Router
           (⇅)
            |                     :
           [⇄]-------------------[⇄]
           /:\                   /:\
        PC1 : PC2             PC3 : PC4
            :                     :
     VLAN 1 : VLAN 2       VLAN 1 : VLAN 2

Connect the router to one of the switches as a member of all VLANs to be
routed and make sure all traffic forwarded on the link is tagged.

The set up procedure inside the router varies depending on the network
operating system it runs.   Generally three steps are necessary:

  1. "demux" all VLANs to unique interfaces
  2. set up a gateway address[^1] on each VLAN interface
  3. enable IP forwarding between these interfaces

Exercises:

 - Verify forwarding between VLANs/subnets
 - What is the source IP of the packets on the destination LAN?


[^1]: usually .1 in a /24 network
