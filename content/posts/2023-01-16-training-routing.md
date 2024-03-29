---
title:  Network Training — Routing
author: Joachim Wiberg
date:   2023-01-16 09:58:42 +0100
categories:
 - networking course
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

```shell
          Router
           (⇅)
            |                     :
           [⇄]-------------------[⇄]
           /:\                   /:\
        PC1 : PC2             PC3 : PC4
            :                     :
    VLAN 10 : VLAN 20     VLAN 10 : VLAN 20
```

Connect the router to one of the switches as a member of all VLANs to be
routed and make sure all traffic forwarded on the link is tagged.

Since we are introducing a new network entity into the mix, we must
change the IP address plan slightly for our network, compared to the
previous exercises.  We give the router the .1 address in both LANs.

| **Node** | **VLAN**  |  **Subnet**  | **Address** | **Netmask**   |
|----------|-----------|--------------|-------------|---------------|
| Router   | 10T       | 10.0.10.0/24 | 10.0.10.1   | 255.255.255.0 |
| Router   | 20T       | 10.0.20.0/24 | 10.0.20.1   | 255.255.255.0 |
| PC1      | 10U       | 10.0.10.0/24 | 10.0.10.10  | 255.255.255.0 |
| PC2      | 20U       | 10.0.20.0/24 | 10.0.20.20  | 255.255.255.0 |
| PC3      | 10U       | 10.0.10.0/24 | 10.0.10.30  | 255.255.255.0 |
| PC4      | 20U       | 10.0.20.0/24 | 10.0.20.40  | 255.255.255.0 |

### Setup

The set up procedure inside the router varies depending on the network
operating system it runs, check the documentation or online help for
details. Generally three steps are necessary:

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

```shell
         Brouter
          [⇄⇅⇄]------------------[⇄]
           /:\                   /:\
        PC1 : PC2             PC3 : PC4
            :                     :
    VLAN 10 : VLAN 20     VLAN 10 : VLAN 20
```


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

 1. Set up a rule to DENY ALL forwarding between `vlan10` and `vlan20`
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

As you may have heard, there aren't enough IPv4 addresses to go around.
Though it must have seemed like a stretch back in the 70's, it turns out
that the 4 billion address pool is way too small to allocate an address
for all the stuff we have connected to the internet.

For this reason, your home gateway usually has a single publicly
routable IP address which is shared by all the computers, phones, and
smart toasters connected to it.  This process of sharing an IP address
is known as *masquerading*, and in this exercise we will learn how to
activate it. 
              
```shell
          (Home)              (Internet)
          [⇄⇅⇄]-----------------[⇄⇅⇄]
           / \                   / \
        PC1  PC2              PC3   PC4
    VLAN 10   VLAN 10     VLAN 20   VLAN 20
       10.0.10.0/24           10.0.20.0/24
```

### Exercises

On the *Home Brouter*:

  1. Set PC1 and PC2 to be on the same network (10.0.10.0/24)
  2. Set trunk port ip address (Make sure it is on the same network as trunk port address from the internet brouter, i.e. 1.1.1.1/24)
  3. Enable access to 10.0.20.0/24 network via the trunk port
  4. Set up the masquerading on devices connected to "Home zone" using nftables (prefered over iptables) 
  
On the *Internet Brouter*:
  1. Set PC3 and PC4 to be on the same network (10.0.20.0/24) 
  2. Set trunk port ip address (Make sure it is on the same network as trunk port address from the home brouter, i.e. 1.1.1.2/24)

Verify that:
  1. PC1 can SSH to PC4
  2. PC4 no longer can SSH to PC1 (assuming the rules from the *Basic Firewall* exercise are still present)

### Questions

 - What is the source IP of the packets arriving to PC4 from PC1?
 - What is different from the *Routing Between VLANs* exercise?
 - Why is different, why can't PC1 ping PC4 anymore?
 - Describe what the *Brouter* does: routing and masquerading


## Port Forwarding

We have seen that masquerading allows multiple clients to share a single
public IP address.  Clients can open connections to any server on the
public internet and the gateway is capable of correctly identifying the
return traffic and route it back to the client.

Now lets look at the opposite scenario, where some server exists on the
local network which we want to be reachable from the Internet.  How is
the gateway to know which local device to forward the incoming
connection request to?  This is the reason to intodruce *port forwarding*
rules, which is the focus of this exercise.

### Exercises

On the *Brouter*:

 1. Add a port forwarding rule on home Brouter port 222 to PC1 port 22
 2. Verify that PC4 can SSH to PC1 by connecting to the IP address of
    the Brouter's vlan10, using custom port 222
 3. Verify that both PC1 and PC4 still can SSH to the Brouter using
    their respective gateway addresses (on the default port 22)

### Questions

There are no questions for this exercise, when you get it to work you
are done.  Maybe spend some time thinking how you could explain all
this to someone else.


[0]: https://en.wikipedia.org/wiki/Bridge_router
[Infix]: https://github.com/kernelkit/infix
