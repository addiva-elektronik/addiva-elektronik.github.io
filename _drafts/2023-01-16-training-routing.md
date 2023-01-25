---
title:  Network Training — Routing
author: Joachim Wiberg
date:   2023-01-16 09:58:42 +0100
---

Lab exercises suitable for practicing bridging, VLANs, and routing
between devices on different subnets.

As the router in this set of exercises we recommend using [Infix][], a
Network Operating System (NOS) based on Linux.  It comes with a plain
shell interface supporting basic functions like bridging, routing, and
firewalling.

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

Since we are introducing a new network entity into the mix, we must
change the IP address plan slightly for our network, compared to the
previous exercises.  We give the router the .1 address in both LANs.

| **Node** | **VLAN** | **Subnet**  | **Address** | **Netmask**   |
|----------|----------|-------------|-------------|---------------|
| Router   | 1T       | 10.0.1.0/24 | 10.0.1.1    | 255.255.255.0 |
| Router   | 2T       | 10.0.1.0/24 | 10.0.2.1    | 255.255.255.0 |
| PC1      | 1U       | 10.0.1.0/24 | 10.0.1.10   | 255.255.255.0 |
| PC2      | 2U       | 10.0.2.0/24 | 10.0.2.20   | 255.255.255.0 |
| PC3      | 1U       | 10.0.1.0/24 | 10.0.1.30   | 255.255.255.0 |
| PC4      | 2U       | 10.0.2.0/24 | 10.0.2.40   | 255.255.255.0 |

### Setup

The set up procedure inside the router varies depending on the network
operating system it runs, check the documentation or online help for
details.   Generally three steps are necessary:

 1. "demux" the VLAN trunk to unique (untagged) interfaces
 2. set up a gateway address on each VLAN interface
 3. enable IP forwarding between these interfaces

The routing table on the router should now have a network route to each
of the subnets it is connected to.

The setup procedure for the end devices (PCs):

 1. Set up IP address
 2. Add a gateway to router (new!)

The routing tables on the end devices should now have a default route,
i.e., "for all IPs not on the subnet I'm on, send to .1 on my subnet."

### Exercises & Questions

 - Verify forwarding between VLANs/subnets, e.g., ping PC4 from PC1
 - What is the source IP of the packets on the destination LAN?
 - Describe what the router does


## Collapse Router and Switch (Brouter)

In the previous exercises each network element has been a freestanding
and single-function entity.  In this exercise we will replace our router
and left-hand switch with a [*Brouter*][0] using [Infix][].

> When using GNS3, follow the instructions in the Infix README on how to
> install it as a GNS3 appliance.

         Brouter
          [⇄⇅⇄]------------------[⇄]
           /:\                   /:\
        PC1 : PC2             PC3 : PC4
            :                     :
     VLAN 1 : VLAN 2       VLAN 1 : VLAN 2


### Exercises

 1. Start by placing all (connected) ports in a VLAN aware bridge
 2. Verify connectivity between, e.g., PC1 and PC3
 3. With bridging in place we can set up VLAN interfaces on top of the bridge
 4. Set up IP addresses on each interface, same IP address plan as above
 5. Enable IP forwarding between the interfaces
 6. Verify forwarding between VLANs/subnets, e.g., ping PC4 from PC1


## Basic Firewall

The point of this exercise is to extend the *Brouter* with a basic firewall.

  > The safest way to approach firewalling as a beginner; always start
  > by blocking everything (DENY ALL) and then add rules that open up
  > only for certain flows (ALLOW) -- keep it small so can see it all.

Two main tools exist today to manage the Linux netfilter stack: the
legacy `iptables` and nftables (`nft`).  The one recommended for new
deployments is nftables.


### Exercises

On the *Brouter*:

 1. Set up a rule to DENY ALL forwarding between `vlan1` and `vlan2`
 2. Add a rule to allow only SSH/TCP in both directions
 3. Verify SSH login from, e.g., PC1 to PC4 works
 4. Verify other traffic is blocked, e.g., ping from PC1 to PC4

### Questions

 1. Which tool did you use, why?  If you used iptables, consider redoing
    the exercise with nftables instead.
 2. How can you inspect the active kernel firewall rules?
 3. How can you make the rules persistent across reboots?

Describe your setup and the answers to the questions to the course
leader, or write them down and send them in.


## IP Masquerading

What most users call their home "router" is actually a *Brouter* and IP
masquerading firewall.  In this exercise we learn how to activate the
masquerading.

### Exercises

On the *Brouter*:

  1. Set up the masquerading such that all devices in VLAN 2 are the
     "LAN zone" and all devices in VLAN 1 are in the "Internet zone"
  2. Verify that PC4 can SSH to PC1
  3. Verify that PC1 no longer can SSH to PC4 (despite the rules from
     the *Basic Firewall* exercise)

### Questions

 - What is the source IP of the packets arriving to PC1 from PC4?
 - What is different from the *Routing Between VLANs* exercise?
 - Why is different, why can't PC1 ping PC4 anymore?
 - Describe what the *Brouter* does: routing and masquerading


## Port Forwarding

In this last exercise we learn about how *port forwarding* works.

### Exercises

On the *Brouter*:

 1. Add a port forwarding rule on vlan1 port 222 to PC4 port 22
 2. Verify that PC1 can SSH to PC4 by connecting to the IP address of
    the Brouter's vlan1, using custom port 222
 3. Verify that both PC1 and PC4 still can SSH to the Brouter using
    their respective gateway addresses (on the default port 22)

### Questions

There are no questions for this exercise, when you get it to work you
are done.  Maybe spend some time thinking how you could explain all
this to someone else.


[0]: https://en.wikipedia.org/wiki/Bridge_router
[Infix]: https://github.com/kernelkit/infix
