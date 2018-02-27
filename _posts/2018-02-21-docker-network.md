---
layout: default
title: Docker networks
---
# Docker networks

Docker has several options for connecting containers to the network.

- The default is **bridge**. The container is connected to a Linuxbridge named docker0 via a veth pair.
- **host**: Connects the container directly to the host's network.
- **overlay**: Allows networks that span container hosts.
- **macvlan**: Generates a unique MAC address for the container; useful for porting applications that expect direct connection to a physical network.
- **none**: The container is not connected to the network at all.

## Exploring Docker's default network

See also the network tutorial on the Docker documentation site.

Docker creates a Linuxbridge named docker0, to which containers are connected by default. Let's start two containers and explore their network connections:

{% highlight shell %}
$ docker run -dit --name alpine1 alpine ash
$ docker run -dit --name alpine2 alpine ash
{% endhighlight %}

List the networks on the host:

{% highlight shell %}
$ brctl show
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242991b7644       no              veth3220a8b
                                                        veth44e00db
$ ip a
...
5: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:99:1b:76:44 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:99ff:fe1b:7644/64 scope link
       valid_lft forever preferred_lft forever
11: veth44e00db@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether 96:f0:99:f1:73:61 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::94f0:99ff:fef1:7361/64 scope link
       valid_lft forever preferred_lft forever
13: veth3220a8b@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether 7e:87:30:44:be:ca brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::7c87:30ff:fe44:beca/64 scope link
       valid_lft forever preferred_lft forever
{% endhighlight %}


Docker has created two veth interfaces, one per container, that are plugged into docker0. 
veth interfaces come in pairs. In the above example, the other end of each pair is, respectively, interface 10 and 12. The ip command can't display the other ends because they reside in their own network namespaces. They are visible in the containers:

{% highlight shell %}
$ docker attach alpine1
/ # ip a
...
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
{% endhighlight %}


This is interface 10, the partner of interface 11 (also known as veth44e00db) on the host.

The docker network inspect command displays detailed information about the current network setup:

{% highlight shell %}
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
b69732400cfd        bridge              bridge              local
edd08488095e        host                host                local
fccec8d809d7        none                null                local
{% endhighlight %}

Docker has set up three networks: Bridge, host and none. We are interested in the bridge network.
{% highlight shell %}
$ docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "b69732400cfd0b055bf263d9af26613bf669c9b11a9edd166738ca355ba38861",
        "Created": "2018-02-20T21:47:06.666421795-05:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
{% endhighlight %}

This shows us subnet and gateway of the bridge network.
{% highlight shell %}
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "0e4a4cbeb752fd16e5faa1e459dafea312fbb62459497ef57125abb98e273891": {
                "Name": "alpine1",
                "EndpointID": "758ebe2a8b88321d7afb568b3d80df6e25b929b82b6350ff026f635a1c3f31e0",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "97186683aaa47dbae6ef13e6e5c3c5860b62b025a76d56f408f556e8e1bcc021": {
                "Name": "alpine2",
                "EndpointID": "c7e0b34ab67fe786d429f1bcc20dc926b9006d736e297472d0e78537186b34c9",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
        },
{% endhighlight %}

Two containers are currently plugged into the bridge. Docker reports their addresses, which can be 
compared with (and should be identical to) the IP addresses reported inside the containers.
{% highlight shell %}
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
{% endhighlight %}


## Exploring a user-defined bridge network

To create another bridge network:

{% highlight shell %}
$ docker network create --driver bridge alpine-net
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
d2afe0681bab        alpine-net          bridge              local
ba349b2efdd7        bridge              bridge              local
edd08488095e        host                host                local
fccec8d809d7        none                null                local
$ docker network inspect alpine-net
[
    {
        "Name": "alpine-net",
        "Id": "d2afe0681bab080e5a1983f16d6ff9eac156685cc2b0668a60302669165d06da",
        "Created": "2018-02-21T21:19:50.27901365-05:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
     ...
]
{% endhighlight %}


Subnet and gateway addresses differ from the addresses in the *bridge* network.

{% highlight shell %}
$ brctl show
bridge name     bridge id               STP enabled     interfaces
br-d2afe0681bab         8000.02424d9bd9b3       no
docker0         8000.0242022630ae       no              veth108e2d2
                                                        veth8812dfc
virbr0          8000.52540050a64e       yes             virbr0-nic
{% endhighlight %}


The new network is implemented by another Linuxbridge, whose name is derived from the ID of alpine-net.

To connect a container to the new network, use the --network option, for example:

{% highlight shell %}
$ docker run -dit --name alpine3 --network alpine-net alpine ssh
{% endhighlight %}


Or connect a running container:

{% highlight shell %}
$ docker network connect alpine-net alpine2
{% endhighlight %}


This adds alpine-net to alpine2, which is now connected to two networks:

{% highlight shell %}
$ docker attach alpine2
/ # ip a
...
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
11: eth1@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth1
       valid_lft forever preferred_lft forever
{% endhighlight %}


### Automatic service discovery on user-defined networks

On user-defined networks, container can use the names of other containers for connecting. 
The alpine2 container is connected to both networks *bridge* and *alpine-net*. 
It has connectivity to alpine1 on *bridge*, but only when using its IP address 172.17.0.2. 

{% highlight shell %}
/ # ping 172.17.0.2 -c3
PING 172.17.0.2 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.286 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.275 ms
64 bytes from 172.17.0.2: seq=2 ttl=64 time=0.180 ms

--- 172.17.0.2 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.180/0.247/0.286 ms
/ # ping alpine1
ping: bad address 'alpine1'
{% endhighlight %}


On the other hand,
alpine3, which is connected to user-defined *alpine-net*, can be reached by name:

{% highlight shell %}
/ # ping alpine3 -c3
PING alpine3 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.219 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.199 ms
64 bytes from 172.18.0.3: seq=2 ttl=64 time=0.163 ms

--- alpine3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.163/0.193/0.219 ms
{% endhighlight %}


## Host network

A container connected to a host network doesn't have a network namespace. It uses the network resources from the host.

As an example, run a web server in a container connected to the host network, 
and inspect the network configuration on the host.

{% highlight shell %}
$ docker run --rm -dit --network host --name webserver nginx
$ docker inspect --format '{{ .State.Pid }}' webserver
1614
$ sudo ss -tlnp |grep 1614
LISTEN     0      128    *:80      *:*    users:(("nginx",pid=2421,fd=6),("nginx",pid=2404,fd=6))
$ ip a
$ brctl show
{% endhighlight %}


The contained nginx process is simply connected to the network. No network namespace is used. 
No additional network interface or bridge has been created.
