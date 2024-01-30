---
title:  "Using OSPF Unnumbered Interfaces with Infix Linux (revisited)"
author: Jon-Olov Vatn
date:   2024-01-30 17:06:42 +0200
tags:
 - linux
 - networking
 - infix
 - ospf
---

> This post updates an earlier [blog
> post](../../../../2023/06/13/using-ospf-unnumbered-interfaces-with-infix-linux-and-frr/)
> where a combination of Infix CLI, Linux and FRR commands were used
> to accomplish this use case. Since then, Infix has developed to and
> this new blog post accomplishes the same use case solely with Infix
> CLI commands. As Infix continues to develop, it is recommended to
> visit [Infix
> documentation](https://github.com/kernelkit/infix/tree/main/doc)
> pages.


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
3. In OSPF configuration there are two settings (done per OSPF area)
   - enable OSPF on intended interfaces 
   - set Ethernet interfaces as OSPF *point-to-point* networks `
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

### Setting hostname (R1, R2, etc.)

By default, Infix units are assigned the name `switch-12-34-56` where
the last part (`12-34-56`) matches the last 3 octets of the unit's
base MAC address. In the examples below, we change the hostname to R1,
R2, etc. To change it, login to the unit (console or SSH), and issue
`cli` command to enter [Infix][infix] CLI. Then change the hostname as
shown below.

    admin@infix-04-00-00:~$ 
    admin@infix-04-00-00:~$ cli
    admin@infix-04-00-00:/> configure 
    admin@infix-04-00-00:/config/> set system hostname R1
    admin@infix-04-00-00:/config/> leave
    admin@R1:/> copy running-config startup-config 
    admin@R1:/>

### Configuring IP address (and enabling IP forwarding)

Below we show how to set the address on R1 using via [Infix][infix] CLI
As the unit is intended to work as IPv4 router, ensure IP forwarding
is enabled on the Ethernet interface(s).

    admin@R1:/> 
    admin@R1:/> configure
    admin@R1:/config/> set interface lo ipv4 address 10.1.1.1 prefix-length 32
    admin@R1:/config/> set interface eth0 ipv4 address 10.1.1.1 prefix-length 32
    admin@R1:/config/> set interface eth0 ipv4 forwarding true

To view the changes done so far, use the `diff` command.

    admin@R1:/config/>
    admin@R1:/config/> diff
    interfaces {
      interface eth0 {
    +    ipv4 {
    +      forwarding true;
    +      address 10.1.1.1 {
    +        prefix-length 32;
    +      }
    +    }
      }
      interface lo {
        ipv4 {
    +      address 10.1.1.1 {
    +        prefix-length 32;
    +      }
        }
      }
    }
    admin@R1:/config/>

Apply changes by leaving the configuration context and store changes
to startup

    admin@R1:/config/> 
    admin@R1:/config/> leave
    admin@R1:/> copy running-config startup-config 
    admin@R1:/>

Status of IP address assignment can be viewed using `show interfaces` command.

    admin@R1:/> 
    admin@R1:/> show interfaces 
    INTERFACE       PROTOCOL   STATE       DATA                                     
    eth0            ethernet   UP          0c:ec:d1:04:00:00                        
                    ipv4                   10.1.1.1/32 (static)
                    ipv6                   fe80::eec:d1ff:fe04:0/64 (link-layer)
    eth1            ethernet   DOWN        0c:ec:d1:04:00:01                        
    eth2            ethernet   DOWN        0c:ec:d1:04:00:02                        
    eth3            ethernet   DOWN        0c:ec:d1:04:00:03                        
    eth4            ethernet   DOWN        0c:ec:d1:04:00:04                        
    eth5            ethernet   DOWN        0c:ec:d1:04:00:05                        
    eth6            ethernet   DOWN        0c:ec:d1:04:00:06                        
    eth7            ethernet   DOWN        0c:ec:d1:04:00:07                        
    eth8            ethernet   DOWN        0c:ec:d1:04:00:08                        
    eth9            ethernet   DOWN        0c:ec:d1:04:00:09                        
    lo              ethernet   UP          00:00:00:00:00:00                        
                    ipv4                   127.0.0.1/8 (static)
                    ipv4                   10.1.1.1/32 (static)
                    ipv6                   ::1/128 (other)
    admin@R1:/>


 
### OSPF configuration

OSPF configuration takes place under the *routing,
control-plane-protocol and ospfv2* subcontext. In line with the IETF
OSPF YANG model, an OSPF instance name has to be given. The name has
local significance only; here the instance is called
*default*. Enabling OSPF per interface and configuring interface-type
are done within OSPF area context (here the backbone area 0.0.0.0 is
used).

    admin@R1:/config/> 
    admin@R1:/config/> edit routing control-plane-protocol ospfv2 name default
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface lo enabled true
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth0 enabled true
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth0 interface-type point-to-point 
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/>

Changes can be shown with the `diff` command.

    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> diff	
    +routing {
    +  control-plane-protocols {
    +    control-plane-protocol ospfv2 name default {
    +      ospf {
    +        areas {
    +          area 0.0.0.0 {
    +            interfaces {
    +              interface eth0 {
    +                interface-type point-to-point;
    +                enabled true;
    +              }
    +              interface lo {
    +                enabled true;
    +              }
    +            }
    +          }
    +        }
    +      }
    +    }
    +  }
    +}
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
    admin@R1:/>


### Checking connectivity

Assuming the above configuration is done on R1 and R2 (with address
10.1.1.2), we can verify connectivity, by pinging the neighbor
router. 

    admin@R1:/> 
    admin@R1:/> ping 10.1.1.2
    PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
    64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=1.21 ms
    64 bytes from 10.1.1.2: icmp_seq=2 ttl=64 time=1.54 ms
    ^C
    --- 10.1.1.2 ping statistics ---
    2 packets transmitted, 2 received, 0% packet loss, time 1002ms
    rtt min/avg/max/mdev = 1.205/1.374/1.544/0.169 ms
    admin@R1:/>

We can also view the IP routing table using the `show routes` command

    admin@R1:/> 
    admin@R1:/> show routes 
    PREFIX                        NEXT-HOP                          PREF  PROTOCOL  
    10.1.1.2/32                   10.1.1.2                            20  ospf      
    admin@R1:/>

OSPF status can be shown with commands such as `show ospf neighbor`,
`show ospf interface`, `show ospf routes`, etc.

    admin@R1:/> 
    admin@R1:/> show ospf neighbor 

    Neighbor ID     Pri State           Up Time         Dead Time Address         Interface                        RXmtL RqstL DBsmL
    10.1.1.2          1 Full/-          1m01s             38.568s 10.1.1.2        eth0:10.1.1.1                        0     0     0

    admin@R1:/> show ospf routes 
    ============ OSPF network routing table ============
    N    10.1.1.1/32           [0] area: 0.0.0.0
                               directly attached to lo
    N    10.1.1.2/32           [10] area: 0.0.0.0
                               via 10.1.1.2, eth0

    ============ OSPF router routing table =============

    ============ OSPF external routing table ===========

    admin@R1:/>




## Troubleshooting

If OSPF fails to setup the routes, it is usually good to start
checking from bottom up. Is cable connected, are Ethernet interfaces
up, etc. To troubleshoot connectivity, it can be hard to use `IPv4
ping` as we have not assigned any IPv4 address to `eth0` other than
the /32 address. Pinging the neighbor relies on OSPF to work when
unnumbered interfaces are used.

Instead we can use *tcpdump* to listen for LLDP packets.

    admin@R1:/> tcpdump eth0
    tcpdump: listening on eth0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    07:22:13.051412 LLDP, length 232: R2
    07:22:31.856495 LLDP, length 232: R1
    07:22:43.063845 LLDP, length 232: R2

    3 packets captured
    3 packets received by filter
    0 packets dropped by kernel
    ^C
    admin@R1:/>

In this case, R1 could see both its outgoing LLDP message and R2's
incoming LLDP. A good sign of connectivity, but we ought to have seen
some OSPF signalling too.

We can also use IPv6 ping to *all-hosts* address. Below R1 sends such
a ping on interface eth0.


    admin@R1:/> ping interface eth0 ff02::1
    ping: Warning: source address might be selected on device other than: eth0
    PING ff02::1(ff02::1) from :: eth0: 56 data bytes
    64 bytes from fe80::eec:d1ff:fe04:0%eth0: icmp_seq=1 ttl=64 time=0.022 ms
    64 bytes from fe80::e47:2dff:fe15:0%eth0: icmp_seq=1 ttl=64 time=1.34 ms
    64 bytes from fe80::eec:d1ff:fe04:0%eth0: icmp_seq=2 ttl=64 time=0.057 ms
    64 bytes from fe80::e47:2dff:fe15:0%eth0: icmp_seq=2 ttl=64 time=2.97 ms
    ^C
    --- ff02::1 ping statistics ---
    2 packets transmitted, 2 received, +2 duplicates, 0% packet loss, time 1002ms
    rtt min/avg/max/mdev = 0.022/1.097/2.968/1.203 ms
    admin@R1:/>

By looking at sequence numbers, we see duplicate responses for each
ping; one from R1 itself and one from R2. This is also a good sign of
connectivity. 

If neighbor connectivity work but you are unable to ping to 10.1.1.2
from R1, the following hints for troubleshooting is suggested.

- Check the IP address status (`show interfaces`) and/or IP address
  and forwarding configuration (`show running-config`, look for
  *interface eth0* and *interface lo*)
- Check OSPF status, in particular `show ospf neighbor` and `show ospf
  interfaces` for hints.
- Chec OSPF configuration, either by `show running-config` (look for
  *ietf-routing:routing*) or by entering configuration mode as shown below.

        admin@R1:/> configure 
        admin@R1:/config/> edit routing control-plane-protocol ospfv2 name default 
        admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> show
        ospf {
          areas {
            area 0.0.0.0 {
              interfaces {
                interface eth0 {
                  interface-type point-to-point;
                  enabled true;
                }
                interface lo {
                  enabled true;
                }
              }
            }
          }
        }
        admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/>



## Larger setup

To go from the simple setup with two nodes ([Figure 2](#fig2) to the
larger in [Figure 1](#fig1), we should include the remaining
interfaces on R1 and R2, and the do the corresponding configuration
for R3-R5.

On R1, we need include eth1 for IP settings and OSPF.

Doing IP setting via Infix CLI is shown below (eth1 should get same
config as eth0 got before).

First setting IPv4 address and enable forwarding

    admin@R1:/> 
    admin@R1:/> configure 
    admin@R1:/config/> set interface eth1 ipv4 address 10.1.1.1 prefix-length 32
    admin@R1:/config/> set interface eth1 ipv4 forwarding true

Then OSPF configuration

    admin@R1:/config/> 
    admin@R1:/config/> edit routing control-plane-protocol ospfv2 name default
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 enabled true 
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> set ospf area 0.0.0.0 interface eth1 interface-type point-to-point

OSPF configuration can be inspected

    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> show
    ospf {
      areas {
        area 0.0.0.0 {
          interfaces {
            interface eth0 {
              interface-type point-to-point;
              enabled true;
            }
            interface eth1 {
              interface-type point-to-point;
              enabled true;
            }
            interface lo {
              enabled true;
            }
          }
        }
      }
    }
    admin@R1:/config/routing/control-plane-protocol/ospfv2/name/default/> leave
    admin@R1:/>


Corresponding IP and OSPF configuration are then applied to all
routers in [Figure 1](#fig1).  OSPF is able to establish routes to all
routers. Here is the result at R1.

    admin@R1:/> show ospf routes 
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


    admin@R1:/>


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


On R1, *eth2* is configured with IP address 10.0.1.1/24, and enabled
for OSPF within area 0.0.0.0. The eth2 interface is kept as (regular)
broadcast interface (not point-to-point).

    admin@R1:/> 
    admin@R1:/> configure 
    admin@R1:/config/> set interface eth2 ipv4 address 10.0.1.1 prefix-length 24
    admin@R1:/config/> set interface eth2 ipv4 forwarding true
    admin@R1:/config/> set routing control-plane-protocol ospfv2 name default ospf area 0.0.0.0 interface eth2 enabled true 
    admin@R1:/config/> leave
    admin@R1:/>

If corresponding setup is done on R6, OSPF would now establish the
following routing table

    admin@R1:/> show ospf routes 
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


    admin@R1:/>


Verify be letting PC1 ping PC2

    PC1> ping 10.0.6.26

    84 bytes from 10.0.6.26 icmp_seq=1 ttl=60 time=2.536 ms
    84 bytes from 10.0.6.26 icmp_seq=2 ttl=60 time=2.386 ms
    84 bytes from 10.0.6.26 icmp_seq=3 ttl=60 time=2.629 ms
    ^C
    PC1>

Done!

[infix]: https://github.com/kernelkit/infix
[infix-hybrid]: https://github.com/kernelkit/infix#hybrid-mode
[frr]: http://docs.frrouting.org
