---
title:  "Setting-up a NETCOF-Client-to-Infix-Server Communication"
date:   2023-05-12 11:30:40 +0100
author: Emir Hasanovic
categories:
 - networking course
---

This blog provides an explanation on how to set-up the basic communication between a NETCONF client and the [Infix](https://github.com/kernelkit/infix) server. It contains three sections which give us some answers on the following questions:

  - How to install Infix server?
  - How to set-up a NETCONF client?
  - How to start the client-server communication?

Additionally, there is a list of known issues and workarounds for the used tools. 
<!-- more -->

## Installation of Infix server

There are two ways for installing Infix (with already set-up server on it) mentioned in this section: Infix on GNS3 emulator & Infix on [qeneth](https://github.com/wkz/qeneth).

**Infix on GNS3**

After [installing GNS3](https://docs.gns3.com/docs/getting-started/installation/linux/) to your device, it is required to import Infix as an appliance, as explained on [Infix github page](https://github.com/kernelkit/infix): 

>Download the [latest build](https://github.com/kernelkit/infix/releases/tag/latest) of the x86_64, or x86_64_classic flavor. Unpack in a dedicated directory and use ["Import Appliance"](https://docs.gns3.com/docs/using-gns3/beginners/import-gns3-appliance/) to install the .gns3a file into GNS3. Infix (x86_64) is in the "Router" category, it has 10 interfaces available by default for use as switch ports or routing. The classic build only has one interface by default, geared more towards acting as an end device.

It is important to mention to install Infix version without the "classic" tag in the name since those versions don't support NETCONF. 

After the Infix is added to GNS3, it is possible to create a simple setup as below: 

                   .--.-.
                  ( (    )__
                 (_,  \ ) ,_)  NAT1
                   '-'--`--'
                     |
                     |
                    (⇅)
                   Infix

It is only left to assing an IP address to the Infix device, either a static one by: 

     ip addr add 10.10.10.1/24 dev eth0

or a dynamic one by using DHCP server: 

     udhcpc -i eth0

After that your NETCONF server is ready for use! 


**Infix on qeneth**

Follow the instuctions on the [github page](https://github.com/wkz/qeneth) to install qeneth on your machine. 
Create a separate working directory in which you want to run your qeneth topology and download the Infix image for installing the server, for example:  
```
~$ mkdir tutorial
~$ cd tutorial/
~/tutorial$ wget -q https://github.com/kernelkit/infix/releases/download/latest/infix-x86_64.tar.gz
~/tutorial$ tar xf infix-x86_64.tar.gz infix-x86_64/infix-x86_64.img
~/tutorial$ mv infix-x86_64/infix-x86_64.img .
~/tutorial$ rmdir infix-x86_64/
~/tutorial$ rm infix-x86_64.tar.gz
```

After that it is required to create a topology file, i.e.: ~/tutorial/topology.dot.in
In this case we only need an Infix instance connected to the host system, so the input of topology file is as simple as this: 

```
graph "host-net" {
        node [shape=record];
        qn_template="infix-x86_64";

        host   [label="host  | { <tap-infix> tap-infix }"];
        infix  [label="infix | { <eth0> eth0 | <eth1> eth1 }"];

        host:"tap-infix" -- infix:eth0;
}
```

The next step is to generate topology and start qeneth. After that it is only left to set some ip addresses to the TAP interface and to Infix instance:

```
~/host-net$ qeneth generate && qeneth start
Info: Generating topology
Info: Generating node YAML
gvpr: warning: Using value of uninitialized edge attribute "qn_headport" of "host--infix"
Info: Generating executables
Info: Launching infix
~/host-net$ ip -br link | grep tap-infix
tap-infix        DOWN           ce:fe:82:f8:cd:19 <BROADCAST,MULTICAST>
~/host-net$ ip addr add 10.10.10.1/24 dev tap-infix
~/host-net$ qeneth console infix
Trying 127.0.0.1...
Connected to localhost.

Infix by KernelKit (console)
infix login: root
Note: use help, show, and setup commands to set up and diagnose the system.
root@infix:~# ip addr add 10.10.10.2/24 dev eth0
root@infix:~# ping -c 3 10.10.10.1
PING 10.10.10.1 (10.10.10.1): 56 data bytes
64 bytes from 10.10.10.1: seq=0 ttl=64 time=0.241 ms
64 bytes from 10.10.10.1: seq=1 ttl=64 time=0.474 ms
64 bytes from 10.10.10.1: seq=2 ttl=64 time=0.504 ms

--- 10.10.10.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.241/0.406/0.504 ms
root@infix:~#
telnet> q
Connection closed.
}
```





## Installation of NETCONF client

**netopeer2**

**netconf-client**

## Basic server-client communication

## Known issues and workarounds
>- netopeer installation errors and warnings
>- netopeer removal 
>- netopeer problems with ssh keys (not a common problem)
>- qeneth limitation to one instance at a time


