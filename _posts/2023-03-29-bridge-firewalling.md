---
title:  "Bridge Firewalling"
author: Joachim Wiberg
date:   2023-03-29 17:06:42 +0200
categories: linux networking bridge firewall
---

Most current Linux distributions (2021) have bridge firewalling enabled
by default.  Particularly in Ubuntu, since 21.04, this has been known to
cause a fair amount of head scratching!

<!-- more -->

This blog post shows only how to rectify the problem using the `sysctl`
command.  Another blog post may venture into managing bridge firewall
rules with either the traditional `ebtables` command, or the new `nft`
suite.  Example:

```
$ sudo ebtables -L
Bridge table: filter

Bridge chain: INPUT, entries: 0, policy: ACCEPT

Bridge chain: FORWARD, entries: 0, policy: ACCEPT

Bridge chain: OUTPUT, entries: 0, policy: ACCEPT
$ sudo nft list tables bridge
$
```

## Fixing the Problem

The relevant kernel bridge settings can be seen with the following
command:

```
$ sysctl net.bridge
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-filter-pppoe-tagged = 0
net.bridge.bridge-nf-filter-vlan-tagged = 0
net.bridge.bridge-nf-pass-vlan-input-dev = 0
```

The culprits are the first three enabled bridge netfilter settings.
They enable hooks in the bridge for trapping and dropping frames on
different layers.

We create the file `/etc/sysctl.d/90-bridge-no-filter.conf` by using a
clever HERE script:

```
$ sudo tee /etc/sysctl.d/90-bridge-no-filter.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables  = 0
net.bridge.bridge-nf-call-arptables = 0
EOF
```

Activate the new settings by restarting the systemd service, or
rebooting your system:

```
$ sudo systemctl restart systemd-sysctl.service
```

Verify that the new settings took:

```
$ sysctl net.bridge
net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-ip6tables = 0
net.bridge.bridge-nf-call-iptables = 0
...
```

For more information on this topic, see the following libVirt wiki page:
https://wiki.libvirt.org/page/Net.bridge.bridge-nf-call_and_sysctl.conf
