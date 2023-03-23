---
title:  "Sysrepo vs Clixon"
date:   2023-02-17 10:00:00 +0100
author: Joachim Wiberg
categories: netconf restconf infix
---

When composing a new [Network Operating System (Infix)][1] several key
decisions have to be made.  Infix is based on [Buildroot][2], a very
well maintained embedded Linux distribution, which in itself provide
only the bare necessities to create and maintain a full operating
system.  It comes with native Linux `/etc/` configuration files and a
shell prompt.

Earlier experience have shown most customers rarely have the expertise
to handle embedded Linux systems this way, much less configure and set
up intricate networking services.  Nor should they have to!  Providing a
user-friendly command line interface (CLI) and/or web interface is a key
selling point.  However, manually maintaining modeling and syntax is
very hard and time consuming, not to mention difficult to get right.

Therefore, we decided early on to build on the work done by IEEE and
IETF in modeling common settings and behavior of networking devices.
Although fairly comprehensive, they provide a *complete and coherent*
view of a switch or router.  It is our job to make it user-friendly!


## Criteria

Setting out to find a suitable Open Source component to fill this void
we had the following requirements ([RFC2119][3]):

 - Must use [YANG][4] for modeling
 - Must support [NETCONF][5] for remote management
 - Must have good documentation
 - Should be an active/maintained project
 - Should support [RESTCONF][6]
 - Should have at least rudimentary command line tools
 - Optional support for [CORECONF][7]

## Evaluation

While many alternatives exist, the two contenders in our evaluation are
[Sysrepo][8] and [Clixon][9].  A basic comparison of features shows them
on sort of equal pairing:

| **Feature**   | **Sysrepo**   | **Clixon** |
|---------------|---------------|------------|
| Data store    | ✔             | ✔          |
| YANG          | libyang       | native     |
| NETCONF       | netopeer2     | native     |
| Documentation | ✔             | ✔          |
| Maintained    | ✔             | ✔          |
| RESTCONF      | -             | native     |
| CLI           | netopeer2-cli | autocli    |
| CORECONF      | -             | -          |

Both projects have traction and is in active use.  For example, Deutsche
Telekom have built sysrepo plugins for:

 - [IETF System model](https://github.com/telekom/sysrepo-plugin-system/)
 - [IETF Interfaces model](https://github.com/telekom/sysrepo-plugin-interfaces/)

So initially we were leaning towards Sysrepo, much thanks to existing
plugins.  However, Clixon's native support for RESTCONF (read+write) and
the excellent, from YANG models, automatically generated CLI is what in
the end tilted the scales in favor of Clixon.  Sure, sysrepo has the
upcoming [Rousette](https://github.com/CESNET/rousette) plugin but that
is not yet finished.  We want to spend our time building value for our
customers, so the more of the basic framework-y things we can get out
of the box and not have to worry about, the better.

We've already started an implementation of the IETF System model in
Clixon using [Augeas](https://augeas.net/) to modify files in `/etc` and
next up we'll do IETF Interfaces and start looking into the IEEE 802.1Q
bridging model as well.  For the initial drops we will not need much
more than that for Infix.


## Honorable Mention

The [OpenCPE](https://github.com/opencpe) project, with freenetconfd,
mand, etc. [for OpenWRT](https://openwrt.org/inbox/howto/opencpe) is
a very interesting alternative for those looking into using OpenWRT
as a base for their product.


## About AEL

At Addiva Elektronik we specialize in networking and power solutions.
Both from a hardware and embedded software perspective, offering our
customers a from "farm to table" experience.

We have strategic partnerships with both Marvell and Microchip and have
extensive experience building managed Ethernet switches for demanding
industrial applications.

For more information, see: <https://addiva.se>

[1]: https://github.com/kernelkit/infix
[2]: https://buildroot.org/
[3]: https://www.ietf.org/rfc/rfc2119.txt
[4]: https://www.rfc-editor.org/rfc/rfc6020
[5]: https://www.rfc-editor.org/rfc/rfc6241
[6]: https://www.rfc-editor.org/rfc/rfc8040
[7]: https://datatracker.ietf.org/doc/draft-ietf-core-comi/
[8]: https://www.sysrepo.org/
[9]: https://www.clicon.org/
