---
title:  "Firewall Container"
author: Joachim Wiberg
date:   2024-03-12 08:08:41 +0100
tags:
 - infix
 - container
 - containers
 - networking
 - firewall
 - docker
 - podman
---

{{% figure src="/images/docker.webp" link="https://www.docker.com/resources/what-container/"
    width="200" caption="Docker on your switch/router? Yes!" class="right-floated" %}}

This is the third post about containers in Infix, now it's time to dive
into using containers as "apps" in Infix for custom firewall setups.
Infix currently does not natively support setting up firewalls or ACLS,
so the approach shown here is useful even for more basic setups.  See
the [first post][1] for a background and networking basics.

> **Note:** this post assumes knowledge and familiarity with the [Infix
> Network Operating System](https://kernelkit.github.io/).  Ensure you
> have a network connection to your Infix system and can log in to it
> using SSH.  For help, see the [networking documentation][0].

----

## Introduction

A clever feature of containers in Linux is that any namespace[^1] used
to "contain" a process can be either unique to the process, a set of
processes sharing the same container, or shared between containers.
Infix limits this flexibility for containers to:

 1. Fully contained, e.g., unique network namespace, fully isolated
 2. Shared network namespace with the host

The second one is what we'll focus on in this blog post because it
allows us to add functionality to Infix that extends its networking
capabilities.

The application in question is an `nftables` container, again from the
[curiOS][2] project, that we will use to create basic router setup with
NAT[^2] firewall.

{{% figure src="/images/basic-router.svg" caption="Basic router, single WAN port, LAN ports bridged." class="center" %}}


## Configuration

The Infix configuration consists of two parts: networking setup and the
container.  We start with the networking, we want a single port as our
WAN port, connected to the Internet, all the other ports are bridged
in a LAN configuration.

```
admin@infix:/> configure
admin@infix:/config/> set dhcp-client client-if eth0
admin@infix:/config/> edit interface br0
admin@infix:/config/interface/br0/> set ipv4 address 192.168.0.1 prefix-length 24
admin@infix:/config/interface/br0/> end
admin@infix:/config/> set interface eth1 bridge-port bridge br0
admin@infix:/config/> set interface eth2 bridge-port bridge br0
admin@infix:/config/> set interface eth3 bridge-port bridge br0
admin@infix:/config/> leave
```

Time for the container configuration.  This time we don't need to mark
any interfaces as `container-network`.  The key configuration settings
are `set network host`, which tells Infix to share its network namespace
with the container, and *privileged* mode.

```
admin@infix:/> configure
admin@infix:/config> edit container nftables
admin@infix:/config/container/nftables/> set image docker://ghcr.io/kernelkit/curios-nftables:24.02.0
admin@infix:/config/container/nftables/> set network host
admin@infix:/config/container/nftables/> set privileged true
admin@infix:/config/container/nftables/> edit mount nftables.conf
admin@infix:/config/container/nftables/mount/nftables.conf/> set target /etc/nftables.conf
admin@infix:/config/container/nftables/mount/nftables.conf/> text-editor content
... interactive editor starts up where you can paste your rules ...
admin@infix:/config/container/nftables/mount/nftables.conf/> leave
```

{{% figure src="/images/firewall.svg" link="https://wiki.nftables.org/wiki-nftables/index.php/Main_Page"
    caption="nftables is Linux firewall" width="200" class="right-floated" %}}

Notice the command `text-editor content`, it opens up an interactive
[text editor][4] where you can paste in the contents of the file
`/etc/nftables.conf` that will be mounted in the container.

The `text-editor` command base64 encodes the content and stores it
in the Infix `startup-config`, thus ensuring that the configuration
of the firewall is retained even when reconfigure the container, or
upgrade the container image.

> See below for an example `nftables.conf` to paste in.


## The Result

We should now have a running container.

```
admin@infix:/> show container 
CONTAINER ID  IMAGE                                      COMMAND               CREATED         STATUS         PORTS       NAMES
aae7178c00f8  ghcr.io/kernelkit/curios-nftables:24.02.0  /usr/sbin/nft-hel...  56 seconds ago  Up 56 seconds              nftables
```

This particular container cannot be connected to, like in the previous
post, because it is only a plain application container with no embedded
shell included.  But we can exit to our host's shell and inspect the
kernel netfilter rules:

```
admin@infix:/> exit
admin@infix:~$ sudo nft list ruleset
table ip filter {
	chain wan {
		icmp type echo-request limit rate 5/second accept
	}

	chain lan {
		icmp type echo-request accept
		ip protocol . th dport vmap { tcp . 22 : accept, tcp . 53 : accept, udp . 53 : accept, udp . 67 : accept }
	}

	chain input {
		type filter hook input priority filter; policy drop;
		ct state vmap { invalid : drop, established : accept, related : accept }
		iifname vmap { "eth0" : jump wan, "lo" : accept, "br0" : jump lan }
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state vmap { invalid : drop, established : accept, related : accept }
		iifname "br0" accept
	}
}
table ip nat {
	chain prerouting {
		type nat hook prerouting priority filter; policy accept;
	}

	chain postrouting {
		type nat hook postrouting priority srcnat; policy accept;
		ip saddr 192.168.0.0/24 oif "eth0" masquerade
	}
}
```

> Here we run the `nft` program shipped with Infix.  It is of course
> possible to run the `nft` binary from the container, albeit not as
> easily:
>
> ```shell
> admin@infix:~$ sudo podman run --network=host --privileged=true --entrypoint=/usr/sbin/nft ghcr.io/kernelkit/curios-nftables:24.02.0 list ruleset
> ```


## `/etc/nftables.conf`

This is the example firewall configuration referred to above.
If your interface names or IP address plan differs, make sure
to modify the file to match.

```
define WAN = eth0
define LAN = br0
define NET = 192.168.0.0/24

# Drop all rules to allow reloading this file after edit
flush ruleset

table ip filter {
    chain wan {
        # Accept ping for diagnostics, with rate limit
        icmp type echo-request limit rate 5/second accept

        # allow SSH connections from some well-known internet host
        #ip saddr 81.209.165.42 tcp dport ssh accept
    }

    chain lan {
        icmp type echo-request accept

        # allow DHCP, DNS and SSH from the private network
        ip protocol . th dport vmap {
            tcp . 22 : accept,
            udp . 53 : accept,
            tcp . 53 : accept,
            udp . 67 : accept
        }
    }

    chain input {
        type filter hook input priority 0; policy drop;

        # Allow traffic from established and related packets, drop invalid
        ct state vmap {
            established : accept,
            related     : accept,
            invalid     : drop
        }

        # allow loopback traffic, anything else jump to chain for further evaluation
        iifname vmap {
            lo   : accept,
            $WAN : jump wan,
            $LAN : jump lan
        }

        # the rest is dropped by the above policy
    }

    chain forward {
        type filter hook forward priority 0; policy drop;

        # Allow traffic from established and related packets, drop invalid
        ct state vmap {
            established : accept,
            related     : accept,
            invalid     : drop
        }

        # connections from LAN to the internet or to other intranets are allowed
        iifname $LAN accept

        # the rest is dropped by the above policy
    }
}

table ip nat {
    chain prerouting {
        type nat hook prerouting priority filter; policy accept;
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr $NET oif $WAN masquerade
    }
}
```


[^1]: Linux namespaces partition system resources such that one set of
    processes sees one set of resources, while another set of processes
    sees a different set of resources.  There are many types: mount,
	user, PID, network, ...

[^2]: NAT firewall, in this context an RFC 2636 NAPT, or IP masquerading
    firewall with filtering of incoming traffic.  For more information, see
    the excellent [Wikipedia article][3] on the topic.

[0]: https://github.com/kernelkit/infix/blob/main/doc/networking.md
[1]: /2024/03/08/docker-containers-in-infix/
[2]: https://github.com/kernelkit/curiOS/
[3]: https://en.wikipedia.org/wiki/Network_address_translation
[4]: https://github.com/kernelkit/infix/blob/main/doc/cli/text-editor.md
