---
title:  "Setting-up a NETCOF Client-to-Server Communication"
date:   2023-05-12 11:30:40 +0100
author: Emir Hasanovic
categories:
 - networking course
---

This blog provides an explanation on how to set-up the basic communication between a NETCONF client and a NETCONF server installed on [Infix](https://github.com/kernelkit/infix) system. It contains three sections which give us some answers on the following questions:

  - How to install Infix with NETCONF server?
  - How to set-up a NETCONF client?
  - How to start the client-server communication?

Additionally, there is a list of known issues and workarounds for the used tools. 
<!-- more -->

## Installation of Infix with NETCONF server

There are two ways for installing Infix (with already set-up NETCONF server on it) mentioned in this section: Infix on GNS3 emulator & Infix on [*qeneth*](https://github.com/wkz/qeneth).

**Infix on GNS3**

After [installing GNS3](https://docs.gns3.com/docs/getting-started/installation/linux/) to your device, it is required to import Infix as an appliance, as explained on [Infix github page](https://github.com/kernelkit/infix): 

>Download the [latest build](https://github.com/kernelkit/infix/releases/tag/latest) of the *x86_64*, or *x86_64_classic* flavor. Unpack in a dedicated directory and use ["Import Appliance"](https://docs.gns3.com/docs/using-gns3/beginners/import-gns3-appliance/) to install the *.gns3a* file into GNS3. Infix (*x86_64*) is in the "Router" category, it has 10 interfaces available by default for use as switch ports or routing. The classic build only has one interface by default, geared more towards acting as an end device.

It is important to mention that for this purpose it is required to install Infix version without the "classic" tag in the name since those versions don't support NETCONF. 

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

     > ip addr add 10.10.10.1/24 dev eth0

or a dynamic one by using DHCP server: 

     > udhcpc -i eth0

After that your NETCONF server is ready for use! 


**Infix on qeneth**

Follow the instuctions on the [github page](https://github.com/wkz/qeneth) to install *qeneth* on your machine. 
Create a separate working directory in which you want to run your *qeneth* topology and download the Infix image for installing the server, for example:  
```
     ~$ mkdir tutorial
     ~$ cd tutorial/
     ~/tutorial$ wget -q https://github.com/kernelkit/infix/releases/download/latest/infix-x86_64.tar.gz
     ~/tutorial$ tar xf infix-x86_64.tar.gz infix-x86_64/infix-x86_64.img
     ~/tutorial$ mv infix-x86_64/infix-x86_64.img .
     ~/tutorial$ rmdir infix-x86_64/
     ~/tutorial$ rm infix-x86_64.tar.gz
```

After that it is required to create a topology file, i.e.: *~/tutorial/topology.dot.in*.

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

The next step is to generate topology and start *qeneth*. After that it is only left to set some IP addresses to the TAP interface and to Infix instance:

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
```

## Installation of NETCONF client

To set-up a NETCONF client two different packages are proposed in this section: *netopeer2-cli* and *netconf-client*.

**netopeer2**

It should be possible to install *netopeer2* as every other Ubuntu package by running `apt install netopeer2`. If not, it is possible to clone the repository and install it in that way: 
```
     ~$ git clone https://github.com/CESNET/netopeer2.git
     ~$ cd netopeer2
     ~$ mkdir build && cd build && cmake .. && make && make install
```
There could be some missing dependencies so before installing *netopeer2*, it is recommended to check and/or install the following packages:
```
     ~$ sudo apt-get update
     ~$ sudo apt-get install git cmake build-essential bison flex libpcre3-dev libev-dev libavl-dev libprotobuf-c-dev protobuf-c-compiler swig python-dev lua5.2 pkg-config libpcre++-dev openssl libssl-dev libcrypto++-dev zlib1g-dev
```
**netconf**

This package represents a Python-based tool and can be used for setting up both NETCONF servers and NETCONF clients. More details on the actual implementation can be found on [its github page](https://github.com/choppsv1/netconf), as well as in the available [documentation](https://netconf.readthedocs.io/en/master/).

Since the package is based on Python as a prerequisite it is needed to install *Python PIP package* and then use the same to install netconf.
```
      ~$ apt install python3-pip
      ~$ pip install netconf
```
## Basic server-client communication

After we set up the NETCONF server based on Infix (either on GNS3 or *qeneth*) and a client (either with *netopeer* or *netconf*), it is possible to start NETCONF communication and issue different commands for manipulating server configuration or fetching its data. 

**netopeer2-cli**

To start *netopeer* client type `netopeer2-cli`. After CLI is started, it is possible to connect to the host by issuing the following command and accepting authenticity of the host. It will also ask for the user password which in case of *root* user in Infix is empty. 

     > connect --host 192.168.122.102 --login root
     The authenticity of the host '10.10.10.2' cannot be established.
     ssh-rsa key fingerprint is e5:78:fe:c0:38:14:d2:9e:fa:32:1d:ff:06:63:42:14:7b:0f:d8:b3.
     Are you sure you want to continue connecting (yes/no)? yes
     root@192.168.122.102 password: 

Now when the connection is established it is possible to check the status of connection and issue different NETCONF commands. It is possible to explore the available commands with `help` command. 

Below is an example of how to fetch the sytem data (i.e. hostname) from the server, and then how to change change hostname inside the basic configuration. It is just needed to create appropriate *.xml* file and to define it as a new config. 

For example, to change the hostname, create a *hostname.xml* with the following input: 

```
     <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
      <hostname>addiva</hostname>
     </system>
```

To disconnect from the server just send `disconnect` command.

     ~$ netopeer2-cli
     > help
     Available commands:
       auth            Manage SSH authentication options
       knownhosts      Manage the user knownhosts file
       cert            Manage trusted or your own certificates
       crl             Manage Certificate Revocation List directory
       outputformat    Set the output format of all the data
       searchpath      Set the search path for models
       verb            Change verbosity
       version         Print Netopeer2 CLI version
       disconnect      Disconnect from a NETCONF server
       status          Display information about the current NETCONF session
       connect         Connect to a NETCONF server
       listen          Wait for a Call Home connection from a NETCONF server
       quit            Quit the program
       help            Display commands description
       editor          Set the text editor for working with XML data
       cancel-commit   ietf-netconf <cancel-commit> operation
       commit          ietf-netconf <commit> operation
       copy-config     ietf-netconf <copy-config> operation
       delete-config   ietf-netconf <delete-config> operation
       discard-changes ietf-netconf <discard-changes> operation
       edit-config     ietf-netconf <edit-config> operation
       get             ietf-netconf <get> operation
       get-config      ietf-netconf <get-config> operation
       kill-session    ietf-netconf <kill-session> operation
       lock            ietf-netconf <lock> operation
       unlock          ietf-netconf <unlock> operation
       validate        ietf-netconf <validate> operation
       subscribe       notifications <create-subscription> operation
       get-schema      ietf-netconf-monitoring <get-schema> operation
       get-data        ietf-netconf-nmda <get-data> operation
       edit-data       ietf-netconf-nmda <edit-data> operation
       establish-sub   ietf-subscribed-notifications <establish-subscription> operation
       modify-sub      ietf-subscribed-notifications <modify-subscription> operation
       delete-sub      ietf-subscribed-notifications <delete-subscription> operation
       kill-sub        ietf-subscribed-notifications <kill-subscription> operation
       establish-push  ietf-subscribed-notifications <establish-subscription> operation with ietf-yang-push augments
       modify-push     ietf-subscribed-notifications <modify-subscription> operation with ietf-yang-push augments
       resync-sub      ietf-yang-push <resync-subscription> operation
       user-rpc        Send your own content in an RPC envelope
       timed           Time all the commands (that communicate with a server) from issuing an RPC to getting a reply
       ?               Display commands description
       exit            Quit the program
     > connect --host 192.168.122.102 --login root
     The authenticity of the host '192.168.122.102' cannot be established.
     ssh-rsa key fingerprint is 6f:9a:5c:02:70:88:ec:19:76:1a:89:73:9b:74:83:95:b0:44:7b:00.
     Are you sure you want to continue connecting (yes/no)? yes
     root@192.168.122.102 password:
     > status
     Current NETCONF session:
       ID          : 1
       Host        : 10.10.10.2
       Port        : 830
       Transport   : SSH
       Capabilities:
	     urn:ietf:params:netconf:base:1.0
	     urn:ietf:params:netconf:base:1.1
	     urn:ietf:params:netconf:capability:writable-running:1.0
	     urn:ietf:params:netconf:capability:candidate:1.0
	     urn:ietf:params:netconf:capability:confirmed-commit:1.1
	     urn:ietf:params:netconf:capability:rollback-on-error:1.0
	     urn:ietf:params:netconf:capability:validate:1.1
	     urn:ietf:params:netconf:capability:startup:1.0
	     urn:ietf:params:netconf:capability:xpath:1.0
	     urn:ietf:params:netconf:capability:with-defaults:1.0?basic-mode=explicit&also-supported=report-all,report-all-tagged,trim,explicit
	     urn:ietf:params:netconf:capability:notification:1.0
	     urn:ietf:params:netconf:capability:interleave:1.0
	     urn:ietf:params:netconf:capability:url:1.0?scheme=http,https,ftp,ftps,file
	     urn:ietf:params:xml:ns:yang:ietf-yang-metadata?module=ietf-yang-metadata&revision=2016-08-05
	     urn:ietf:params:xml:ns:yang:1?module=yang&revision=2022-06-16
	     urn:ietf:params:xml:ns:yang:ietf-inet-types?module=ietf-inet-types&revision=2013-07-15
	     urn:ietf:params:xml:ns:yang:ietf-yang-types?module=ietf-yang-types&revision=2013-07-15
	     urn:ietf:params:xml:ns:yang:ietf-netconf-acm?module=ietf-netconf-acm&revision=2018-02-14
	     urn:ietf:params:netconf:capability:yang-library:1.1?revision=2019-01-04&content-id=8
	     urn:sysrepo:plugind?module=sysrepo-plugind&revision=2022-08-26
	     urn:ietf:params:xml:ns:netconf:base:1.0?module=ietf-netconf&revision=2013-09-29&features=writable-running,candidate,confirmed-commit,rollback-on-error,validate,startup,url,xpath
	     urn:ietf:params:xml:ns:yang:ietf-netconf-with-defaults?module=ietf-netconf-with-defaults&revision=2011-06-01
	     urn:ietf:params:xml:ns:yang:ietf-netconf-notifications?module=ietf-netconf-notifications&revision=2012-02-06
	     urn:ietf:params:xml:ns:yang:ietf-netconf-monitoring?module=ietf-netconf-monitoring&revision=2010-10-04
	     urn:ietf:params:xml:ns:netmod:notification?module=nc-notifications&revision=2008-07-14
	     urn:ietf:params:xml:ns:netconf:notification:1.0?module=notifications&revision=2008-07-14
	     urn:ietf:params:xml:ns:yang:ietf-x509-cert-to-name?module=ietf-x509-cert-to-name&revision=2014-12-10
	     urn:ietf:params:xml:ns:yang:iana-crypt-hash?module=iana-crypt-hash&revision=2014-08-06
	     urn:ietf:params:xml:ns:yang:iana-if-type?module=iana-if-type&revision=2023-01-26
	     urn:ietf:params:xml:ns:yang:ietf-system?module=ietf-system&revision=2014-08-06&features=ntp,ntp-udp-port,timezone-name&deviations=kernelkit-infix-deviations
     > get-config --source running --filter-xpath /system
     DATA
     <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
       <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
          <hostname>default</hostname>
       </system>
     </data>

     > 
     > edit-config --target running --config=/home/emir/Desktop/hostname.xml
     OK
     > get-config --source running --filter-xpath /system
     DATA
     <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
       <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
         <hostname>addiva</hostname>
       </system>
     </data>
     > disconnect




**netconf-client**

*netconf-client* works more in the request-response manner since it requires host with user and password combination with each request. One of the simplest requests that can be sent to server is to fetch all available configuration data: 
```
     ~$  netconf-client --host infix.local -u root -p '' --get
```
Which returns the output similar to the below one: 
```
 <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
  <interfaces xmlns="urn:ietf:params:xml:ns:yang:ietf-interfaces">
    <interface>
      <name>lo</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:softwareLoopback</type>
      <admin-status>up</admin-status>
      <oper-status>up</oper-status>
      <if-index>1</if-index>
      <ipv4 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <address>
          <ip>127.0.0.1</ip>
          <prefix-length>8</prefix-length>
        </address>
      </ipv4>
      <ipv6 xmlns="urn:ietf:params:xml:ns:yang:ietf-ip">
        <address>
          <ip>::1</ip>
          <prefix-length>128</prefix-length>
        </address>
      </ipv6>
    </interface>
    <interface>
      <name>eth0</name>
      <type xmlns:ianaift="urn:ietf:params:xml:ns:yang:iana-if-type">ianaift:ethernetCsmacd</type>
      <admin-status>up</admin-status>
      <oper-status>up</oper-status>
      <if-index>2</if-index>
    </interface>
  </interfaces>
  ...
  ...
  ...
```

It is also possible to edit configuration, in a similar way as it is done with *netopeer2-cli*. The only difference is that the *.xml* file needs to be wrapped inside ``<config ... </config>`` so it looks as following: 

```
<config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
    <hostname>addiva2</hostname>
  </system>
</config> 
```

Then we issue the following *netconf-cli* commands to set and read configuration: 

```
     ~$ netconf-client --host 192.168.122.102 -u root -p "" --edit-config -i /home/emir/Desktop/hostname.xml
     <rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" nc:message-id="0">
       <ok/>
     </rpc-reply>

     ~$ netconf-client --host 192.168.122.102 -u root -p "" --get-config
     <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" 
      ...
      ...
      ...
      <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
        <hostname>addiva2</hostname>
      </system>
     </data>

```

## Known issues and workarounds

**Errors and warnings after installing netopeer2**

After installing *netopeer2* you may see some errors as the ones below. 

```
     ~$ sudo apt install netopeer2
     The following packages will be INSTALLED:
       netopeer2 
     0 packages upgraded, 1 newly installed, 0 reinstalled, 0 to remove and 10 not upgraded.
     Need to get 212 kB of archives. After unpacking 0 B will be used.
     Get: 1 http://ftp.acc.umu.se/ubuntu jammy/universe amd64 netopeer2 amd64 2.0.35-1ubuntu1 [212 kB]
     Fetched 212 kB in 0s (1 807 kB/s) 
     (Reading database ... 671332 files and directories currently installed.)
     Preparing to unpack .../netopeer2_2.0.35-1ubuntu1_amd64.deb ...
     Unpacking netopeer2 (2.0.35-1ubuntu1) over (2.0.35-1ubuntu1) ...
     Setting up netopeer2 (2.0.35-1ubuntu1) ...
     sysrepoctl error: Invalid option or missing argument: -a
     For more details you may try to increase the verbosity up to "-v3".
     dpkg: error processing package netopeer2 (--configure):
      installed netopeer2 package post-installation script subprocess returned error exit status 1
     Processing triggers for man-db (2.10.2-1) ...
     Errors were encountered while processing:
      netopeer2
     E: Sub-process /usr/bin/dpkg returned an error code (1)
     Setting up netopeer2 (2.0.35-1ubuntu1) ...
     sysrepoctl error: Invalid option or missing argument: -a
     For more details you may try to increase the verbosity up to "-v3".
     dpkg: error processing package netopeer2 (--configure):
      installed netopeer2 package post-installation script subprocess returned error exit status 1
     Errors were encountered while processing:
      netopeer2
```

The *netopeer2-cli* is usable after this, but the warnings persist in the package system. 

To disable warnings from the package manager follow the next workaround:
- Edit the file /var/lib/dpkg/info/netopeer2.postinst
```
    ~$ sudo vim /var/lib/dpkg/info/netopeer2.postinst
```
- Add exit 0 at the very top of the file, so it looks something like this:

```
     #!/bin/sh
     # postinst script for netopeer2
     #
     # see: dh_installdeb(1)

     # addiva: /usr/share/netopeer2/setup.sh is not updated in this release
     # addiva: so it tries using old syntax for sysrepoctl that fails.
     # addiva: With this early `exit 0` we skip the server setup.
     exit 0

     set -e
```
- Tell the package manager to try to fix the broken package(s):
```
    ~$ sudo apt-get -f install
```
>Note: this only fixes the package manager issue, so the server side of the netopeer2 package is still broken. But it works for our purposes since the server side is on Infix anyway and we only really need the CLI.

**Deinstallation issues with netopeer2**

Besides installation errors, *netopeer2* also returns errors when being removed. Therefore, it's impossible to remove the package by just `sudo apt remove netopeer2`. In order to completely uninstall *netopeer2* and avoid future warnings, it is needed to forcefully remove it: 

```
     ~$ sudo mkdir /usr/share/netopeer2
     ~$ sudo mg /usr/share/netopeer2/remove.sh
     ~$ sudo chmod +x /usr/share/netopeer2/remove.sh
     ~$ sudo dpkg --remove --force-all netopeer2
```


**Connection errors with netopeer2**

After the first-time connection with NETCONF server on Infix is closed (either by disconnecting regularly or forcefully), you might experience issues with connecting again to the same Infix instance. 

     > disconnect
     > connect --host infix.local --login root
     nc ERROR: Starting the SSH session failed (kex error : no match for method server host key algo: server [rsa-sha2-512,rsa-sha2-256], client [ssh-rsa,ssh-ed25519,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256,ssh-dss]).
     cmd_connect: Connecting to the infix.local:830 as user "root" failed.

Currently, the only workaround for this is to remove the Infix instace from the *~/.ssh/known_hosts* file on the client side. 


**qeneth limitation**

The current version of *qeneth* allows only one topology to be run at a time. If you try to run a second instance it will not be possible to provide the needed ports and start *qeneth*. In that case you might see something as following: 

```
     ~$ qeneth status
      NODE           PID  CNSOL  MONTR
      infix         lost  10000  10001
      ~$ ./infix
      qemu-system-x86_64: -monitor telnet::10001,server=on,wait=off: Failed to find an available port: Address already in use
```
Therefore, it is important to make sure that all previously running topoligies are closed, before starting Infix server on *qeneth*.
Since *qeneth* runs *qemu* it is easy to explore if there is a running instance in the background by running `ss -ltp`.
To return PID numbers of running *qemu* instances run `pidof qemu-system-x86_64`. After that you need to make sure that you're not running *qemu* in other programs and to kill only the ones that are associated with *qeneth* : `kill <pid>`.


