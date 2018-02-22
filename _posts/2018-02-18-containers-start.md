---
layout: post
title: "Starting with Linux Containers"
date: 2018-02-18
---
## What are containers?

Linux containers are often named light-weight virtual machines. 
I think this is a misleading term. 
What made me understand the nature of containers is the following observation:

> > A *virtual machine* runs in a single process, e.g. qemu. What happens inside
> > this machine is invisible to users on the host.
> > 
> > A *container* is not a single process. Processes that run in a container are 
> > regular processes on the host, and they can be listed with a simple ps command.

While a container's processes are visible on the container host, they are not 
in other containers running on the same host. Obviously, processes on the host that
don't run in a container are not visible either.

How is the separation of containers from each other and from the host accomplished?
The Linux kernel has a number of facilities that help shielding containers, notably 
*namespaces*. 

Here is a definition by Stephane Graber, who use to be one of the maintainers of 
the Linux Containers project (https://stgraber.org/2012/05/04/lxc-in-ubuntu-12-04-lts):

> > LXC is a userspace tool controlling the kernel namespaces and cgroup features to create system or application containers

## What are namespaces?

Useful resources for learning about namespaces are the namespaces(7) manual page and 
[a series of articles](https://lwn.net/Articles/531114) 
published in LWM in 2013, with sample programs.

The Linux kernel provides namespaces for mounts, PIDs, UTS (the hostname), IPC, networking and users.
What are they for? A few examples: 
* A process in the mount namespace can mount and unmount filesystems
without affecting the mount table on the host. 
* A process in the PID namespace can only interact (list, kill etc) with processes in the same namespace. Furthermore, the process IDs in a PID namespace are different from the
process IDs on the host. This way, a container can be moved to a different host and keep its process table intact. Still, a process in a PID namespace also has a process ID on the host. 
* A process in the UTS namespace can change the hostname without affecting the host. This allows associating hostnames with containers.

## How to use namespaces

A process can use options to the clone(2) system call to create a child process in one or more different namespaces.
Another option is the unshare(2) system call, which disconnects the calling process from its current namespace(s) 
and associates them with new ones. The unshare(1) command runs a new process in one or more different namespaces.

## The PID namespace

A PID namespace has a different procfs than the system. If a process is in a PID namespace but
not in a mount namespace, it still sees /proc, where the global procfs is mounted. As a result, 
programs that use /proc (such as the ps command) will report global information.
The PIDs so reported don't have the same meaning in the PID namespace (or, most likely, no meaning at all).

A process cloned into a new PID namespace always has PID 1 there.
