---
title:  "Using OSPF Unnumbered Interfaces with Infix Linux and FRR"
author: Jon-Olov Vatn
date:   2023-06-13 17:06:42 +0200
tags:
 - linux
 - networking
 - infix
 - ospf
---

When using OSPF over Ethernet point to point links, the work to make
IP plans for the PtP connections can be tedious and limits flexibility
as configurations must be coordinated and matched with each neighbor. 
The figure below is used as illustration.

<a name="fig1">**Figure 1**: OSPF network with point-to-point links</a> 

    10.1.1.1     10.1.1.2     10.1.1.4     10.1.1.6
     .----.       .----.       .----.       .----.
     | R1 +-------+ R2 +-------+ R4 +-------+ R6 |
     '---+'       '-+--'       '--+-'       '+---'
          \         |             |         /
           \        |             |        /
            \       |             |       /
             \    .-+--.       .--+-.    /
              `---+ R3 +-------+ R5 +---´
                  '----'       '----'
                10.1.1.3      10.1.1.5

The setup disregards any user networks attached to the routers, and
focus solely on establishing connectivity between routers. (Adding
host subnets is described [further below](#adding-host-subnets)).

With OSPF, router connectivity would be easy to configure if the links
can be treated as *unnumbered point-to-point OSPF interfaces*. With
this approach, each router can be configured individually with minimum
coordination with its neighbours. Ideally, R1 should have IP address
10.1.1.1 assigned to its loopback, advertise 10.1.1.1 via OSPF, and
then connect one port (say *eth0*) towards R2 and another (*eth1*)
towards R3.

In contrast, the custom way is to make an IP plan for each connection,
e.g., 10.0.1.0/30 between R1 and R2, assign address 10.0.1.1/30 to
eth0 on R1 and 10.0.1.2/30 to eth0 on R2. And do the same for subnet
10.0.1.4/30 between R1 and R3, etc. This work we wish to avoid.

> With OSPF unnumbered interfaces we can skip coordinating and
> configuring IP addresses on Ethernet LANs used as point-to-point
> links between routers.

## OSPF unnumbered interfaces in a nutshell

1. Make an IP plan for your routers: In this example R1 is allocated
   the address 10.1.1.1, R2 address 10.1.1.2, etc.
2. Configure the allocated address as /32 to loopback and to all
   Ethernet interfaces intended as unnumbered interfaces.
   On R1, we would configure 10.1.1.1/32 to *lo*, *eth0* and *eth1*. 
3. In OSPF configuration there are two settings
   - advertise the /32 address, e.g, `network 10.1.1.1/32 area 0`
   - set Ethernet interfaces as OSPF *point-to-point* networks
4. Plug the network together.

The routers should detect eachother, build routing tables for
connectivity.


## Minor test: Two nodes

We start out with a simple network, just two nodes

<a name="fig2">**Figure 2**: OSPF network with point-to-point link
    between two routers</a> 
	
    10.1.1.1     10.1.1.2
     .----.       .----.
     | R1 +-------+ R2 |
     '----'       '----'

The first step, the IP plan, is already shown in Figure 2.

### Configuring IP address (and enabling IP forwarding)

Below we show how to set the address on R1, starting with the regular
Linux `ip address` command, then via [Infix][infix] CLI

    root@R1:~$ ip address add 10.1.1.1/32 dev lo
    root@R1:~$ ip address add 10.1.1.1/32 dev eth0
    root@R1:~$	

The result can be viewed:

    root@R1:~$ ip -brief address
    lo               UNKNOWN        127.0.0.1/8 10.1.1.1/32 ::1/128 
    eth0             UP             10.1.1.1/32 fe80::ee5:3bff:fead:0/64 
    eth1             DOWN           
    eth2             DOWN           
    eth3             DOWN           
    root@R1:~$

As the unit is intended to work as IPv4 router, ensure IP forwarding
is enabled.

    root@R1:~$ sysctl -w net.ipv4.ip_forward=1
    net.ipv4.ip_forward = 1
    root@R1:~$

Corresponding using Infix CLI

    root@R1:~$ cli 
    root@R1:exec> configure 
    [edit]
    root@R1:configure> edit interfaces 
    [edit interfaces]
    root@R1:configure> set interface lo ipv4 address 10.1.1.1 prefix-length 32
    [edit interfaces]
    root@R1:configure> set interface eth0 ipv4 address 10.1.1.1 prefix-length 32
    [edit interfaces]
    root@R1:configure> set interface eth0 ipv4 forwarding true
    [edit interfaces]

The resulting interface configuration can be listed

    root@R1:configure> show
    interface eth0 {
      type ethernetCsmacd;
      ipv4 {
        forwarding true;
        address 10.1.1.1 {
          prefix-length 32;
        }
      }
    }
    interface eth1 {
      type ethernetCsmacd;
    }
    interface eth2 {
      type ethernetCsmacd;
    }
    interface eth3 {
      type ethernetCsmacd;
    }
    interface lo {
      type softwareLoopback;
      ipv4 {
        address 10.1.1.1 {
          prefix-length 32;
        }
        address 127.0.0.1 {
          prefix-length 8;
        }
      }
      ipv6 {
        address ::1 {
          prefix-length 128;
        }
      }
    }
    [edit interfaces]
    root@R1:configure> 
 
### OSPF configuration

Here we assume the routers use the [FRR routing stack][frr] (zebra and
ospfd daemons). OSPF support is not yet integrated into the Infix CLI,
thus the FRR CLI is used directly. (See [Infix Hybrid
Mode][infix-hybrid] for information on starting FRR in Infix.)

    root@R1:~$ vtysh 

    Hello, this is FRRouting (version 8.4.2).
    Copyright 1996-2005 Kunihiro Ishiguro, et al.

    R1# configure
    R1(config)# router ospf
    R1(config-router)# network 10.1.1.1/32 area 0
    R1(config-router)# interface eth0
    R1(config-if)# ip ospf network point-to-point 
    R1(config-if)# end
    R1# copy running-config startup-config 
    Building Configuration...
    Integrated configuration saved to /etc/frr/frr.conf
    [OK]
    R1# show running-config 
    Building configuration...

    Current configuration:
    !
    frr version 8.4.2
    frr defaults traditional
    hostname R1
    no ip forwarding
    no ipv6 forwarding
    service integrated-vtysh-config
    !
    interface eth0
     ip ospf network point-to-point
    exit
    !
    router ospf
     network 10.1.1.1/32 area 0
    exit
    !
    end
    R1#


### Checking connectivity

Assuming the above configuration is done on R1 and R2 (with address
10.1.1.2), we can verify connectivity.

We can watch the IP routing table using the Linux `ip route` command

    root@R1:~$ ip route          
    10.1.1.2 nhid 16 via 10.1.1.2 dev eth0 proto ospf metric 20 onlink 
    root@R1:~$

On R2, the corresponding would be

    root@R2:~$ ip route
    10.1.1.1 nhid 16 via 10.1.1.1 dev eth0 proto ospf metric 20 onlink 
    root@R2:~$

More details can be seen in FRR CLI

    root@R1:~$ vtysh 

    Hello, this is FRRouting (version 8.4.2).
    Copyright 1996-2005 Kunihiro Ishiguro, et al.

    R1# show ip ospf route
    ============ OSPF network routing table ============
    N    10.1.1.1/32           [0] area: 0.0.0.0
                               directly attached to lo
    N    10.1.1.2/32           [10] area: 0.0.0.0
                               via 10.1.1.2, eth0

    ============ OSPF router routing table =============

    ============ OSPF external routing table ===========


    R1# show ip ospf neighbor 

    Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
    10.1.1.2          1 Full/-          24.472s           36.561s 10.1.1.2        eth0:10.1.1.1                        0     0     0

    R1# show ip ospf interface 
    eth0 is up
      ifindex 2, MTU 1500 bytes, BW 10000 Mbit <UP,BROADCAST,RUNNING,MULTICAST>
      This interface is UNNUMBERED, Area 0.0.0.0
      MTU mismatch detection: enabled
      Router ID 10.1.1.1, Network Type POINTOPOINT, Cost: 10
      Transmit Delay is 1 sec, State Point-To-Point, Priority 1
      No backup designated router on this network
      Multicast group memberships: OSPFAllRouters
      Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
        Hello due in 4.351s
      Neighbor Count is 1, Adjacent neighbor count is 1
    lo is up
      ifindex 1, MTU 65536 bytes, BW 0 Mbit <UP,LOOPBACK,RUNNING>
      Internet Address 10.1.1.1/32, Broadcast 10.1.1.1, Area 0.0.0.0
      MTU mismatch detection: enabled
      Router ID 10.1.1.1, Network Type LOOPBACK, Cost: 10
      Transmit Delay is 1 sec, State Loopback, Priority 1
      No backup designated router on this network
      Multicast group memberships: <None>
      Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
        Hello due in inactive
      Neighbor Count is 0, Adjacent neighbor count is 0

    R1# show ip route
    Codes: K - kernel route, C - connected, S - static, R - RIP,
           O - OSPF, I - IS-IS, B - BGP, E - EIGRP, T - Table,
           v - VNC, V - VNC-Direct, A - Babel, F - PBR,
           f - OpenFabric,
           > - selected route, * - FIB route, q - queued, r - rejected, b - backup
           t - trapped, o - offload failure

    O   10.1.1.1/32 [110/0] is directly connected, lo, weight 1, 00:03:23
    C * 10.1.1.1/32 is directly connected, eth0, 00:05:05
    C>* 10.1.1.1/32 is directly connected, lo, 00:05:05
    O>* 10.1.1.2/32 [110/10] via 10.1.1.2, eth0 onlink, weight 1, 00:00:48
    R1# ping 10.1.1.2
    PING 10.1.1.2 (10.1.1.2): 56 data bytes
    64 bytes from 10.1.1.2: seq=0 ttl=64 time=0.584 ms
    64 bytes from 10.1.1.2: seq=1 ttl=64 time=0.690 ms
    64 bytes from 10.1.1.2: seq=2 ttl=64 time=0.607 ms
    ^C
    --- 10.1.1.2 ping statistics ---
    3 packets transmitted, 3 packets received, 0% packet loss
    round-trip min/avg/max = 0.584/0.627/0.690 ms
    R1#

## Troubleshooting

If OSPF fails to setup the routes, it is usually good to start
checking from bottom up. Is cable connected, are Ethernet interfaces
up, etc. To verify link connectivity, it can be hard to use `IPv4
ping` as we have not assigned any IPv4 address to `eth0` than the /32
address. Instead we can use *tcpdump* to listen for LLDP packets.

    root@R1:~$ tcpdump -lni eth0
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    12:34:45.853072 IP6 fe80::e31:9ff:fe4b:0 > ff02::2: ICMP6, router solicitation, length 16
    12:34:47.129240 LLDP, length 211: R1
    12:34:47.130563 LLDP, length 211: R2
    ^C
    3 packets captured
    3 packets received by filter
    0 packets dropped by kernel

In this case, R1 could see both its outgoing LLDP message and R2's
incoming LLDP. A good sign of connectivity.

We can also use IPv6 ping to *all-hosts* address. Below R1 sends such
a ping on interface eth0.

    root@R1:~$ ping -6 ff02::1%eth0
    PING ff02::1%eth0 (ff02::1%eth0): 56 data bytes
    64 bytes from fe80::ee5:3bff:fead:0: seq=0 ttl=64 time=0.082 ms
    64 bytes from fe80::e31:9ff:fe4b:0: seq=0 ttl=64 time=1.468 ms (DUP!)
    64 bytes from fe80::ee5:3bff:fead:0: seq=1 ttl=64 time=0.103 ms
    64 bytes from fe80::e31:9ff:fe4b:0: seq=1 ttl=64 time=1.239 ms (DUP!)
    ^C
    --- ff02::1%eth0 ping statistics ---
    2 packets transmitted, 2 packets received, 2 duplicates, 0% packet loss
    round-trip min/avg/max = 0.082/0.723/1.468 ms
    root@R1:~$

We see two responses, one from R1 itself and one from R2. This is also
a good sign of connectivity. (If available, the `-L` ping option can
be used to filter out local responses.)

## Larger setup

To go from the simple setup with two nodes ([Figure 2](#fig2) to the
larger in [Figure 1](#fig1), we should include the remaining
interfaces on R1 and R2, and the do the corresponding configuration
for R3-R5.

On R1, we need include eth1 for IP settings and OSPF.

Doing IP setting via Infix CLI is shown below (eth1 should get same
config as eth0 got before).


    root@R1:~$ cli 
    root@R1:exec> configure 
    [edit]
    root@R1:configure> edit interfaces 
    [edit interfaces]
    root@R1:configure> set interface eth1 ipv4 address 10.1.1.1 prefix-length 32
    [edit interfaces]
    root@R1:configure> set interface eth1 ipv4 forwarding true
    [edit interfaces]

In OSPF, R1 needs to define eth1 as OSPF point-to-point link in FRR CLI.

    R1# configure
    R1(config)# interface eth1
    R1(config-if)# ip ospf network point-to-point 

Resulting OSPF configuration on R1

    R1# show running-config 
    Building configuration...

    Current configuration:
    !
    frr version 8.4.2
    frr defaults traditional
    hostname R1
    no ip forwarding
    no ipv6 forwarding
    service integrated-vtysh-config
    !
    interface eth0
     ip ospf network point-to-point
    exit
    !
    interface eth1
     ip ospf network point-to-point
    exit
    !
    router ospf
     network 10.1.1.1/32 area 0
    exit
    !
    end
    R1#

OSPF is able to establish routes to all routers. Here is the result at
R1 (FRR CLI)

    R1# show ip ospf route
    ============ OSPF network routing table ============
    N    10.1.1.1/32           [0] area: 0.0.0.0
                               directly attached to lo
    N    10.1.1.2/32           [10] area: 0.0.0.0
                               via 10.1.1.2, eth0
    N    10.1.1.3/32           [10] area: 0.0.0.0
                               via 10.1.1.3, eth1
    N    10.1.1.4/32           [20] area: 0.0.0.0
                               via 10.1.1.2, eth0
    N    10.1.1.5/32           [20] area: 0.0.0.0
                               via 10.1.1.3, eth1
    N    10.1.1.6/32           [30] area: 0.0.0.0
                               via 10.1.1.2, eth0
                               via 10.1.1.3, eth1

    ============ OSPF router routing table =============

    ============ OSPF external routing table ===========


    r1#

## Adding host subnets

In addition to the point-to-point links, the router likely has regular
LAN subnets where hosts can be connected. In Figure 3, host subnets
have been added to R1 and R6.

<a name="fig3">**Figure 3**: LAN broadcast networks added at R1 and R6
</a> 

          10.1.1.1     10.1.1.2     10.1.1.4     10.1.1.6
           .----.       .----.       .----.       .----.
     .---. | R1 +-------+ R2 +-------+ R4 +-------+ R6 | .---.
     |PC1| '+--+'       '-+--'       '--+-'       '+--+' |PC2|
     '-+-'  |.1 \         |             |         / .1|  '-+-'
    .45|    |    \        |             |        /    |    |.26
    ---+----+--   \       |             |       /   --+----+--- 
    10.0.1.0/24    \    .-+--.       .--+-.    /    10.0.6.0/24
                    `---+ R3 +-------+ R5 +---´
                        '----'       '----'
                       10.1.1.3     10.1.1.5


On R1, *eth2* is configured with IP address 10.0.1.1/24.

    root@R1:~$ cli 
    root@R1:exec> configure 
    [edit]
    root@R1:configure> edit interfaces 
    [edit interfaces]
    root@R1:configure> set interface eth2 ipv4 address 10.0.1.1 prefix-length 24
    [edit interfaces]
    root@R1:configure> set interface eth2 ipv4 forwarding true
    [edit interfaces

and OSPF configuration in FRR CLI should be updated to advertise network
10.0.1.0/24. *Note:* The eth2 interface is kept as (regular) broadcast
interface (not point-to-point).

    R1# configure
    R1(config)# router ospf
    R1(config-router)# network 10.0.1.0/24 area 0

If corresponding setup is done on R6, OSPF would now establish the
following routing table


    R1# show ip ospf route
    ============ OSPF network routing table ============
    N    10.0.1.0/24           [10] area: 0.0.0.0
                               directly attached to eth2
    N    10.0.6.0/24           [40] area: 0.0.0.0
                               via 10.1.1.2, eth0
                               via 10.1.1.3, eth1
    N    10.1.1.1/32           [0] area: 0.0.0.0
                               directly attached to lo
    N    10.1.1.2/32           [10] area: 0.0.0.0
                               via 10.1.1.2, eth0
    N    10.1.1.3/32           [10] area: 0.0.0.0
                               via 10.1.1.3, eth1
    N    10.1.1.4/32           [20] area: 0.0.0.0
                               via 10.1.1.2, eth0
    N    10.1.1.5/32           [20] area: 0.0.0.0
                               via 10.1.1.3, eth1
    N    10.1.1.6/32           [30] area: 0.0.0.0
                               via 10.1.1.2, eth0
                               via 10.1.1.3, eth1

    ============ OSPF router routing table =============

    ============ OSPF external routing table ===========


    r1#


Verify be letting PC1 ping PC2

    PC1> ping 10.0.6.26
    84 bytes from 10.0.6.26 icmp_seq=1 ttl=60 time=4.670 ms
    84 bytes from 10.0.6.26 icmp_seq=2 ttl=60 time=4.054 ms
    ^C
    PC1>

Done!

[infix]: https://github.com/kernelkit/infix
[infix-hybrid]: https://github.com/kernelkit/infix#hybrid-mode
[frr]: http://docs.frrouting.org
