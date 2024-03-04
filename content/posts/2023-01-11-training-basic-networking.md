---
title:  Network Training — Basic Networking
author: Joachim Wiberg
date:   2023-01-11 16:18:42 +0100
categories:
 - networking course
---

Lab exercises for practicing basic networking with the help of your new
favorite tools.

<!-- more -->

## Install GNS3

The exercises require GNS3, or actual hardware.  E.g., a Marvell
EspressoBIN with a couple of Raspberry Pi as end devices.

If you have hardware already, proceed to the next step.

Installation instructions for GNS3 is available from the [homepage][0],
it is up to you if you want to install it in Linux or Windows, either
should work, but all exercises have been tested with Linux Mint.  The
instructions presented here work for Ubuntu and all its derivatives:

```shell
~$ sudo add-apt-repository ppa:gns3/ppa
~$ sudo apt update
~$ sudo apt install gns3-gui gns3-server xterm wireshark tcpdump
```

Answer the questions you get, allowing regular users to run GNS3.  After
install you need to reboot to start the `gns3-server` and activate your
new permissions.

> **Note:** First time users of GNS3 are recommended to go through the
> tutorial of how to create [Your First GNS3 Topology][1].  Take the
> time to go over all the steps, in particular, learn how to use `ping`
> and take network captures of traffic between network nodes.  You will
> need this in later exercises.


## Static Networking

The GNS3 tutorial is very useful, it basically covers everything in this
first exercise.  For completeness, do this exercise anyway to verify you
do not need a switch.

 - Why don't you need a switch?
 - How does it work?

Set up two devices with a network cable between them.

```shell
     PC1                    PC2
    VPCS ------------------ VPCS
```

 1. Set up device PC1:
    - address: 192.168.1.10
	- netmask: 255.255.255.0
 2. Set up device PC2:
    - address: 192.168.1.20
	- netmask: 255.255.255.0
 3. Verify connectivity with ping


## Dynamic Networking

Now, let's set up a more common example, end devices that use a DHCP
client program to acquire an address "lease" from a DHCP server:

```shell
                   .--.-.
                  ( (    )__
                 (_,  \ ) ,_)  NAT1
                   '-'--`--'
                     |
     PC1             |               PC2
    VPCS -----------[⇄]------------- VPCS
                    SW1
```

The DHCP server functionality is provided by the NAT "cloud", the switch
does not have any such functionality, we only need the switch to allow
more than two devices on the same LAN.

 1. Set up device PC1:
    - DHCP
 2. Set up device PC2:
    - DHCP
 3. Verify connectivity with ping:
    - How do you know which IP address to ping?

With DHCP the end devices not only acquire an IP address and netmask,
they also get a gateway and DNS server.

Is it possible to reach the Internet?  Check with ping if you can
reach the Google DNS (8.8.8.8), or perhaps `ping www.example.com`.


[0]: https://docs.gns3.com/docs/
[1]: https://docs.gns3.com/docs/getting-started/your-first-gns3-topology
