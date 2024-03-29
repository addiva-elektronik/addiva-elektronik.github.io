---
title:   Using NETCONF to Configure a Switch
date:    2023-05-12 11:30:40 +0100
author:  Emir Hasanovic
aliases: /2023/05/12/using-netconf-client-and-server-to-update-switch-configuration/
toc:     true
tags:
 - networking
 - infix
 - netconf
---

This blog post shows how to configure a switch using NETCONF command
line clients.  As an example, we read and set the _hostname_ of a
NETCONF capable switch.

Two command line clients are explored: _netopeer2-cli_, included in the
[Netopeer2][] NETCONF project, and _[netconf-client][]_, which is a
Python client and library.

NETCONF Server we use [Infix][], but any NETCONF capable network switch
should suffice.

Hints on installing the NETCONF clients on Linux as well as Infix as a
virtual switch on the same Linux PC (GNS3 or Qeneth) are provided further
below.

<!-- more -->

## Using a NETCONF Client

We wish to read and set system hostname via NETCONF ([ietf-system.yang][]).


```
 .----. .42              .1 .--------.
 | PC +---------------------+ Switch |
 '----'     10.0.1.0/24     '--------'
 NETCONF                     NETCONF
 Client                      Server
```

### Preparation

First you need the NETCONF login credentials for your NETCONF Server
(the switch).  In case you use Infix[^1], the default credentials are
(user/pass): admin/admin

It is assumed you have installed your NETCONF client[^2].


[^1]: See [hints on installing Infix](#hints-on-installation-of-infix-as-netconf-server).
[^2]: See [hints on installing netopeer2-cli and netconf-client](#installation-of-netconf-client).

### Update config with netopeer2-cli

Start `netopeer2-cli` and connect to the server (the Switch)

```shell
~$ netopeer2-cli
> connect --host 10.0.1.1 --login admin
The authenticity of the host '10.0.1.1' cannot be established.
ssh-rsa key fingerprint is e5:78:fe:c0:38:14:d2:9e:fa:32:1d:ff:06:63:42:14:7b:0f:d8:b3.
Are you sure you want to continue connecting (yes/no)? yes
admin@10.0.1.1 password: admin
```

We can read the whole running config with `get-config --source running`,
but for easier overview, we filter out everything except the config related
to ietf-system.yang.

```xml
> get-config --source running --filter-xpath /system
DATA
<data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
    <hostname>infix</hostname>
  </system>
</data>
```

To change the hostname on the switch to _foo_, create a file on the PC,
e.g., *hostname.xml* with the following input:

```xml
<system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
  <hostname>foo</hostname>
</system>
```

Now we can set the config using `edit-config`, read-out the
configuration to verify, and disconnect.

```xml
> edit-config --target running --config=/path/to/hostname.xml
OK
> get-config --source running --filter-xpath /system
DATA
<data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
    <hostname>foo</hostname>
  </system>
</data>
> disconnect
>
```

<!-- more, perhaps  -->

### Update config with netconf-client

*netconf-client* works more in the request-response manner since it
requires host with user and password combination with each request. One of
the simplest requests that can be sent to server is to fetch all available
configuration data:

```
~$ netconf-client --host 10.0.1.1 -u admin -p admin --get
```
Which returns the output similar to the below one: 

```xml
 <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0">
 ...
 ...
 ...
 <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
   <hostname>infix</hostname>
 </system>
```

It is also possible to edit configuration, in a similar way as it is done
with *netopeer2-cli* above. The only difference is that the *hostname.xml*
file needs to be wrapped inside ``<config ... </config>`` as follows:

```xml
<config xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
    <hostname>foo</hostname>
  </system>
</config> 
```

Then we issue the following *netconf-cli* commands to set and read configuration: 

```xml
~$ netconf-client --host 10.0.1.1 -u admin -p admin --edit-config -i /path/to/hostname.xml
<rpc-reply xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" xmlns:nc="urn:ietf:params:xml:ns:netconf:base:1.0" nc:message-id="0">
  <ok/>
</rpc-reply>

~$ netconf-client --host 10.0.1.1 -u admin -p admin --get-config
 <data xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" 
 ...
 ...
 ...
 <system xmlns="urn:ietf:params:xml:ns:yang:ietf-system">
   <hostname>foo</hostname>
 </system>
</data>
```



## Installation of NETCONF client

To set-up a NETCONF client two different packages are proposed in this
section: *netopeer2-cli* and *netconf-client*.

### netopeer2-cli

The `netopeer2-cli` program is a part of the [Netopeer2][] project.  See
their GitHub page for more information.

It should be possible to install *netopeer2* as every other Ubuntu
package by running `apt install netopeer2`.

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
~$
```

If the package is not available, it is possible to clone its repository
and install it in that way:

```bash
~$ git clone https://github.com/CESNET/netopeer2.git
~$ cd netopeer2
~$ mkdir build && cd build && cmake .. && make && make install
```

There could be some missing dependencies so before installing
*netopeer2*, it is recommended to check and/or install the following
packages:

```bash
~$ sudo apt-get update
~$ sudo apt-get install git cmake build-essential bison flex libpcre3-dev  \
                libev-dev libavl-dev libprotobuf-c-dev protobuf-c-compiler \
				 swig python-dev lua5.2 pkg-config libpcre++-dev openssl   \
				 libssl-dev libcrypto++-dev zlib1g-dev
```

As you could see there are some *error* messages returned after the
installation. It is important to highlight that the package is
successfully installed and can be used, but the warnings persist in the
package system. More instructions on how to get around this can be found
in the last section of this blog.


### Python-based NETCONF client

Since the package is based on Python, as a prerequisite it is needed to
install *Python PIP package* and then use the same to install netconf.

```bash
~$ apt install python3-pip
```

Then we can use `pip` to install the client locally:

```bash
~$ pip install netconf
```

For more information, see the associated [project][netconf-client] page
at GitHub, and their excellent [documentation][netconf-python-doc].


## Hints on installation of Infix as NETCONF server


Infix is a great alternative if you do not have access to NETCONF capable
switch. It enables you to make the following setup _virtually_ within your
Linux PC.
```
 .----. .42              .1 .--------.
 | PC +---------------------+ Switch |
 '----'     10.0.1.0/24     '--------'
 NETCONF                    NETCONF
 Client                     Server
```

Please see the [README][Infix] in the Infix GitHub Repo.  It contains
guidelines for installing Infix and running virtually using
[Qeneth][qeneth-github] or [GNS3][gns3].

> Note, use the *x86_64* version of Infix, **NOT** the *x86_64_classic*
> version, as the latter does not include NETCONF server support

### Infix on GNS3

After [installing GNS3][] to your device, it is required to import Infix
as an appliance, as explained on [Infix][] GitHub page:

> Download the [latest build][] of the *x86_64*, or *x86_64_classic*
> flavor.  Unpack in a dedicated directory and use [Import Appliance][]
> to install the *.gns3a* file into GNS3. Infix (*x86_64*) is in the
> "Router" category, it has 10 interfaces available by default for use
> as switch ports or routing.  The classic build only has one interface
> by default, geared more towards acting as an end device.

It is important to mention that for this purpose it is required to install
Infix version without the "classic" tag in the name since those versions
don't support NETCONF.

After the Infix is added to GNS3, it is possible to create a simple
setup as below:

```
                   .--.-.
                  ( (    )__
                 (_,  \ ) ,_)  NAT1
                   '-'--`--'
                     |
                     |
                    (⇅)
                   Infix
```

By default, the Infix device gets an IPv6 address assigned, which can be
used for setting-up the NETCONF communication. The only requirement is
to have a tap-interface defined to connect between the host device and
Infix device. This is an example of IPv6 address to be used:
`fe80::ff:fe00:0%tap-infix`

In case we want to use IPv4 address, it is only left to assing an IP
address to the Infix device and since the NAT interface is in use, we
obtain one from DHCP server by running:

```
> udhcpc -i eth0
```

The assigned IP address is in form *192.168.122.y*, therefore we can
update the system overview and use this IP address while setting up
client-to-server connection:

```
 .----. .x               .y .--------.
 | PC +---------------------+ Switch |
 '----'  192.168.122.0/24   '--------'
 NETCONF                    NETCONF
 Client                     Server
```


### Infix on qeneth

Follow the instructions on the [github page](https://github.com/wkz/qeneth)
to install *qeneth* on your machine.

The major difference as compared to the tutorial is that the *x86_64* Infix
version should be used (rather than *x86_64_classic*). Thus, as compared to
[Host Interaction Tutorial](https://github.com/wkz/qeneth#host-interaction)
you should:

- download infix-x86_64.tar.gz (see below)
- and modify the 'topology.dot.in' accordingly

Create a separate working directory in which you want to run your *qeneth*
topology and download the Infix image for installing the server, for example:

```shell
~$ mkdir tutorial
~$ cd tutorial/
~/tutorial$ wget -q https://github.com/kernelkit/infix/releases/download/latest/infix-x86_64.tar.gz
~/tutorial$ tar xf infix-x86_64.tar.gz infix-x86_64/infix-x86_64.img
~/tutorial$ mv infix-x86_64/infix-x86_64.img .
~/tutorial$ rmdir infix-x86_64/
~/tutorial$ rm infix-x86_64.tar.gz
```

After that it is required to create a topology file, i.e.:
*~/tutorial/topology.dot.in*.

In this case we only need an Infix instance connected to the host system,
so the input of topology file is as simple as this:

```dot
graph "host-net" {
    node [shape=record];
    qn_template="infix-x86_64";

    host   [label="host  | { <tap0> tap0 }"];
    infix  [label="infix | { <eth0> eth0 | <eth1> eth1 }"];

    host:"tap0" -- infix:eth0;
}
```

On your PC, you will get a tap0 interface, connected to the eth0 of Infix
(running virtually on the PC).

```
 .----. .42                .1 .--------.
 | PC +-----------------------+ Infix  |
 '----'tap0  10.0.1.0/24  eth0'--------'
 NETCONF                      NETCONF
 Client                       Server
```

The next step is to generate topology and start *qeneth*. After that it
is only left to set some IP addresses to the TAP interface and to the
Infix instance (setting IPv4 address to the Infix device is not
mandatory since IPv6 address is assigned to it automatically and can be
used for NETCONF communication).

If you decide to work with IPv6 address, you can find it inside the
infix device:

```shell
root@infix-00-00-00:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group iface qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 02:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::ff:fe00:0/64 scope link
       valid_lft forever preferred_lft forever
```

Or you can find it by pinging the remote Infix device behind the tap
interface and reading the address from reply by running:

```shell
~$ ping -6Lc1 ff02::1%eth0 | awk '/64 bytes from/{sub(/:$/, "", $4); print $4}'
fe80::ff:fe00:0%tap0
```

On the other hand, if you decide to first set an IPv4 address:


```shell
~/tutorial$ qeneth generate && qeneth start
Info: Generating topology
Info: Generating node YAML
gvpr: warning: Using value of uninitialized edge attribute "qn_headport" of "host--infix"
Info: Generating executables
Info: Launching infix

~/tutorial$ ip -br link | grep tap
tap0             DOWN           ce:fe:82:f8:cd:19 <BROADCAST,MULTICAST>

~/tutorial$ ip addr add 10.0.1.42/24 dev tap0
~/tutorial$ qeneth console infix

Infix by KernelKit (console)
infix login: admin
Password: admin

Note: use help, show, and setup commands to set up and diagnose the system.

admin@infix:~# ip addr add 10.0.1.1/24 dev eth0
admin@infix:~# ping -c 3 10.0.1.42
PING 10.0.1.1 (10.0.1.1): 56 data bytes
64 bytes from 10.0.1.42: seq=0 ttl=64 time=0.241 ms
64 bytes from 10.0.1.42: seq=1 ttl=64 time=0.474 ms
64 bytes from 10.0.1.42: seq=2 ttl=64 time=0.504 ms

--- 10.0.1.42 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.241/0.406/0.504 ms
admin@infix:~#
telnet> q
Connection closed.
```

## Known issues and workarounds

### Errors and warnings after installing netopeer2

After installing *netopeer2* you may see some errors as the ones below. 

```shell
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

```shell
~$ sudo vim /var/lib/dpkg/info/netopeer2.postinst
```

- Add exit 0 at the very top of the file, so it looks something like this:

```shell
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

```shell
~$ sudo apt-get -f install
```

> **Note:** this only fixes the package manager issue, so the server
> side of the netopeer2 package is still broken. But it works for our
> purposes since the server side is on Infix anyway and we only really
> need the CLI.

### Deinstallation Issues With Netopeer2

Besides installation errors, *netopeer2* also returns errors when being
removed.  Therefore, it's impossible to remove the package by just

    sudo apt remove netopeer2

In order to completely uninstall *netopeer2* and avoid future warnings,
it is needed to forcefully remove it:

```shell
~$ sudo mkdir /usr/share/netopeer2
~$ sudo mg /usr/share/netopeer2/remove.sh
~$ sudo chmod +x /usr/share/netopeer2/remove.sh
~$ sudo dpkg --remove --force-all netopeer2
```


### Connection Errors with Netopeer2

After the first-time connection with NETCONF server on Infix is closed
(either by disconnecting regularly or forcefully), you might experience
issues with connecting again to the same Infix instance.

```shell
> disconnect
> connect --host infix.local --login admin
nc ERROR: Starting the SSH session failed (kex error :
	no match for method server host key algo:
		server [rsa-sha2-512,rsa-sha2-256],
		client [ssh-rsa,ssh-ed25519,ecdsa-sha2-nistp521,ecdsa-sha2-nistp384,ecdsa-sha2-nistp256,ssh-dss]).
cmd_connect: Connecting to the infix.local:830 as user "admin" failed.
```

Currently, the only workaround for this is to remove the Infix instance
from the `~/.ssh/known_hosts` file on the client side.


### Qeneth Limitations

The current version of *qeneth* allows only one topology to be run at a
time. If you try to run a second instance it will not be possible to
provide the needed ports and start *qeneth*. In that case you might see
something as following:

```
~$ qeneth status
 NODE           PID  CNSOL  MONTR
 infix         lost  10000  10001
~$ ./infix
qemu-system-x86_64: -monitor telnet::10001,server=on,wait=off
Failed to find an available port: Address already in use
```

Therefore, it is important to make sure that all previously running
topoligies are closed, before starting Infix server on *qeneth*.  Since
*qeneth* runs *qemu* it is easy to explore if there is a running
instance in the background by running

    ss -ltp

To return PID numbers of running *qemu* instances run

    pidof qemu-system-x86_64

After that you need to make sure that you're not running *qemu* in other
programs and to kill only the ones that are associated with *qeneth*:

    kill <pid>


## Overview of Netopeer2 CLI commands

```shell
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

> connect --host 10.0.1.1--login admin
The authenticity of the host '10.0.1.1' cannot be established.
ssh-rsa key fingerprint is 6f:9a:5c:02:70:88:ec:19:76:1a:89:73:9b:74:83:95:b0:44:7b:00.
Are you sure you want to continue connecting (yes/no)? yes
admin@10.0.1.1 password: admin

> status
Current NETCONF session:
  ID          : 1
  Host        : 10.0.1.1
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

> disconnect
```


[Netopeer2]:             https://github.com/CESNET/netopeer2
[netconf-client]:        https://github.com/choppsv1/netconf
[netconf-python-doc]:    https://netconf.readthedocs.io/en/master/
[Infix]:                 https://github.com/kernelkit/infix
[latest build]:          https://github.com/kernelkit/infix/releases/tag/latest
[ietf-system.yang]:      https://datatracker.ietf.org/doc/rfc7317/
[qeneth-github]:         https://github.com/wkz/qeneth
[Import Appliance]:      https://docs.gns3.com/docs/using-gns3/beginners/import-gns3-appliance/
[Installing GNS3]:       https://docs.gns3.com/docs/getting-started/installation/linux/
[gns3]:                  https://gns3.com/
