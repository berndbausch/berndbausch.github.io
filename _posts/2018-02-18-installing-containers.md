---
layout: post
title: "How to install Linux Containers"
---

The major distros offer Linux Containers in their repositories. 

Centos, however, only has Linux Containers version 1. Version 2 must be downloaded and compiled. 
To do so, there are a few requirements:
* wget (not strictly necessary, but makes downloading easier)
* gcc, the C compiler
* libvirt

To run Centos containers:

* rsync

To run Ubuntu containers on Centos:

* epel-release
* debootstrap
* perl

Get LXC from https://linuxcontainers.org/downloads/lxc/lxc-2.0.8.tar.gz, 
untar it, configure, make, make install.
