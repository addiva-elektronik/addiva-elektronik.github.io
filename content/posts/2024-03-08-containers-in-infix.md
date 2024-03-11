---
title:  "Docker Containers in Infix"
author: Joachim Wiberg
date:   2024-03-08 15:44:42 +0100
tags:
 - infix
 - container
 - containers
 - docker
 - podman
---

{{% figure src="/images/docker.webp" link="https://www.docker.com/resources/what-container/"
    width="200" caption="Docker on your switch/router? Yes!" class="right-floated" %}}

A network operating system for switches and routers, that runs Docker?

Yes, as of Infix v24.02 support for running containers using [podman][1]
is supported.  Because networking is a first class citizen in Infix, you
can set up quite advanced *virtual topologies* with containers.  This
blog post is the first in a series of posts that aims to show this.

> **Note:** this post assumes knowledge and familiarity with the [Infix
> Network Operating System](https://kernelkit.github.io/).  Ensure you
> have a network connection to your Infix system and can log in to it
> using SSH.  For help, see the [networking documentation][0].

----

## Introduction

All configuration of networking and containers is done through the CLI:

```
admin@example:~$ cli
```

> Notice the slight change in the prompt.  Return to the Bash shell
> using `exit` or Ctrl-D from the "admin-exec" top level of the CLI.


## Networking Basics

{{% figure src="/images/dataplane.svg" width="430" class="right-floated"
 caption="Infix networking, click image for more information."
link="https://github.com/kernelkit/infix/blob/main/doc/networking.md"  %}}

In Infix all network access has to be set up explicitly, so there is no
default container networking setup (it's a security thing).  There are
two types available to choose from:

 - `host`: Ethernet interface
 - `bridge`: Masquerading bridge

The first can be any physical port/interface which is handed over to the
container or, more commonly, one end of a *VETH pair*.

The latter type is usually available as `docker0`, or `podman0`, on your
host system.  These bridges are managed by the container runtime, in the
case of Infix this is podman.  When a container is set to a container
bridge network, a VETH pair is automatically created when the container
is started -- one end is attached to the bridge and the other connected
to the container as a regular interface.

Here's how you create a container bridge:

```
admin@example:/> configure
admin@example:/config> edit interface docker0
admin@example:/config/interface/docker0> set container-network
admin@example:/config/interface/docker0> leave
```


## Web Server Container

{{% figure src="/images/nginx.png" link="https://www.nginx.com" width="60" class="left-floated" %}}

Now, time for a basic web server example.  For our first container we'll
be using [docker://nginx:alpine](https://hub.docker.com/_/nginx).  It's
a relatively small container with the Nginx web server built on top of
the Alpine Linux image.

```
admin@example:/> configure
admin@example:/config> edit container web
admin@example:/config/container/web/> set image docker://nginx:alpine
admin@example:/config/container/web/> edit network
admin@example:/config/container/web/network/> set interface docker0
admin@example:/config/container/web/network/> set publish 8080:80
admin@example:/config/container/web/network/> leave
```

Issuing the command `leave` queues a job to download the image and
create a container in the background.  To see the progress:

```
admin@example:/> show log container
```

or just poll the status command:

```
admin@example:/> show container
CONTAINER ID  IMAGE                           COMMAND               CREATED         STATUS         PORTS               NAMES
c60a6deeea4e  docker.io/library/nginx:alpine  nginx -g daemon o...  2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp  web
```

You should now be able to access the web server on port 8080 of the
host's IP address.

{{% figure src="/images/nginx-welcome.png" class="center" caption="Nginx welcome screen." %}}


## Customizing Content

Deceivingly enough, Docker containers have a thin writable layer that
allows changing just about any file in the image.  The big *HOWEVER*
though is that this layer doesn't survive configuration changes or,
most importantly, image upgrades.

> Use Volumes!

Volumes are a specialized type of "mount", for people familiar with
UNIX systems.  Infix currently supports *named mounts* that provide
a *persistent* writable layer for containers.  Here's how to add one
to your container:

```
admin@example:/> configure 
admin@example:/config/> edit container web 
admin@example:/config/container/web/> edit volume content
admin@example:/config/container/web/volume/content/> set target /usr/share/nginx/html
admin@example:/config/container/web/volume/content/> leave
```

Named volumes have the downside of being opaque to the host, so the
easiest is to upload the content using `scp` or editing it directly
in the container:

```
admin@example:/> container shell web 
d95ce9f7674d:/# vi /usr/share/nginx/html/
50x.html    index.html
d95ce9f7674d:/# vi /usr/share/nginx/html/index.html 
... edit, save & exit from vi ...
d95ce9f7674d:/# 
```


## Container Content in Device Configuration

Save the best for last?  A neat feature is that container content can be
saved in the system's `startup-config` and therefore be automatically be
backed up by administrators snapshotting the system.

> This also means that custom(er) builds of Infix can bundle a built-in
> container's initial configuration in the Infix `factory-config`, which
> can be very useful when deploying at new installations.

This feature is perfectly suited for container applications that need a
specific site setup.  For example a configuration file.  Here we use the
same container image to bundle an `index.html` file:

```
admin@example:/> configure 
admin@example:/config/> edit container web 
admin@example:/config/container/web/> edit mount index.html
admin@example:/config/container/web/mount/index.html/> set target /usr/share/nginx/html/index.html
admin@example:/config/container/web/mount/index.html/> text-editor content
```

The `content` setting is an alternative to `source` for file mounts
which allows providing the contents through the device's configuration.

> The `text-editor` command can be changed in the `system` configuration
> context, but by default it starts a Micro Emacs clone, Mg.

Paste in this:

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to Infix!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to Infix the Network Operating System!</h1>
<p>If you see this page, the nginx web server container has been
installed and is working.</p>

<p>For online documentation and support please refer to the
<a href="https://kernelkit.github.io/">Infix Homepage</a>.<br/>

Commercial support and customer adaptations are available from
<a href="https://addiva.se/electronics">Addiva Elektronik AB</a>.</p>

<p><em>Thank you for reading this blog post!</em></p>
</body>
</html>
```

> Save and exit with the usual Emacs salute: C-x C-x (Ctrl-X Ctrl-c, or
> hold down Ctrl while tapping X and c).

Leave configuration context to activate your changes:

```
admin@example:/config/container/web/mount/index.html/> leave
```

Reload your browser to see the change.


## Fin

That's the end of the first post about containers in Infix.  As usual, remember to

```
admin@example:/> copy running-config startup-config
```

[0]: https://github.com/kernelkit/infix/blob/main/doc/networking.md
[1]: https://podman.io
