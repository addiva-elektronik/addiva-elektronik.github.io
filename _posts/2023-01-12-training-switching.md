---
title:  Network Training — Switching
author: Joachim Wiberg
date:   2023-01-12 14:12:42 +0100
---

Lab exercises suitable for practicing bridging and VLANs.

<!-- more -->

## Connect Two Switches

Starting small, connecting two switches using an "uplink" cable.

        [⇄]--------[⇄]
         |          |
        PC1        PC2

 - Connect the two switches
 - Attach at least one end device to each of the switches
 - Set up static IP addresses on your end devices
 - Start a capture on uplink cable between the switches
 - Start a ping from one device to the other
 - Inspect the capture and verify connectivity


## Basic VLAN Separation

Set up a single switch with VLAN enabled.  One port in VLAN 10 (untagged
member), and two in VLAN 20 (also untagged).

           __:_________
          [__:__ ⇄_____]
           / :   |    \
          /  :   |     \
        PC1  :  PC2    PC3
             :
     VLAN 10 :   VLAN 20
	           :

Set up static IP addresses according the the following table:

| **Node** | **VLAN** |  **Subnet**  | **Address** | **Netmask**   |
|----------|----------|--------------|-------------|---------------|
| PC1      | 10U      | 10.0.10.0/24 | 10.0.10.1   | 255.255.255.0 |
| PC2      | 20U      | 10.0.10.0/24 | 10.0.10.2   | 255.255.255.0 |
| PC3      | 20U      | 10.0.10.0/24 | 10.0.10.3   | 255.255.255.0 |

 - Start a capture on all ports (links between switch and all devices),
   you should have three Wireshark sessions open
 - Verify ping between PC2 and PC3, which ports receive the initial ARP packets?
 - Try ping between PC1 and PC2, it should not work, can you explain
   why?  Where does the ARP packets go?
 

## Switches with VLANs

Now let's add VLANs to the previous exercise. To be able to verify the
port isolation feature of VLANs, we need at least three end devices in
this topology.

            _:_                     _:_
           [_⇄_]-------------------[_⇄_]
           / : \                   / : \
        PC1  :  PC2              PC3 :  PC4
             :                       :
     VLAN 10 :  VLAN 20      VLAN 10 :  VLAN 20

Two ports are untagged members in one VLAN, and at least one is untagged
member in another VLAN.  Make sure the link between the two switches is
a tagged (only) member of both VLANs.

| **Node** | **VLAN** |  **Subnet**  | **Address** | **Netmask**   |
|----------|----------|--------------|-------------|---------------|
| PC1      | 10U      | 10.0.10.0/24 | 10.0.10.1   | 255.255.255.0 |
| PC2      | 20U      | 10.0.20.0/24 | 10.0.20.2   | 255.255.255.0 |
| PC3      | 10U      | 10.0.10.0/24 | 10.0.10.3   | 255.255.255.0 |
| PC4      | 20U      | 10.0.20.0/24 | 10.0.20.4   | 255.255.255.0 |

Members of separate VLANs have different IP subnets, e.g. VLAN 10 has the
10.0.10.0/24 range, and VLAN 20 has 10.0.20.0/24.

 - Verify ping between devices in the same VLAN/subnet
 - Verify that packets on the uplink/trunk have a VLAN tag
   - What is the VLAN priority of packets on the trunk?
   - On which ports can you see the initial ARP of a ping?
 - Verify VLAN separation (no ARP on other VLAN)

