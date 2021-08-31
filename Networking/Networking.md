# Networking

## Contents

- [Switching and Routing](#switching-and-routing)
- [DNS](#dns)
  * [Record Types](#record-types)
  * [nslookup](#nslookup)
  * [dig](#dig)
- [CoreDNS](#coredns)
  * [Configuring a dedicated system as DNS (CoreDNS)](#configuring-a-dedicated-system-as-dns--coredns-)
- [Network Namespaces](#network-namespaces)
  * [Creating network namespaces](#creating-network-namespaces)
  * [Exec in network namespaces](#exec-in-network-namespaces)
  * [Connecting network interfaces](#connecting-network-interfaces)
  * [How enable multiple namespace to inter-communicate](#how-enable-multiple-namespace-to-inter-communicate)
- [Docker Networking](#docker-networking)
- [CNI](#cni)
- [Cluster Networking](#cluster-networking)
- [Pod Networking](#pod-networking)
- [CNI in Kubernetes](#cni-in-kubernetes)
- [CNI Weave](#cni-weave)
- [IP Address Management - Weave](#ip-address-management---weave)
- [Service Networking](#service-networking)
- [DNS in Kubernetes](#dns-in-kubernetes)
- [Core DNS in Kubernetes](#core-dns-in-kubernetes)
- [Ingress](#ingress)
- [Ingress - Annotations and rewrite-target](#ingress---annotations-and-rewrite-target)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## Switching and Routing

Let's assume that we have 2 systems A and B. How will they communicate to each other ??

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-1.svg)

We connect them to a **switch**.
![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-2.svg)

To connect each system to the switch, we need to create some sort of an interface. For which, we use the following commands on both the systems.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-3.svg)

```bash
ip link # to view the switch interface name

ip addr add 192.168.1.10/24 dev eth0 # assign the system A to networking interface

ip addr add 192.168.1.11/24 dev eth0 # assign the system B to networking interface
```

> The switch as of now, only enables communication between the nodes A and B, no external traffic in/out is allowed.

How to connect host B to C both being on different networks ?

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-4.svg)

For this we need a **router**. The router get assigned 2 IPs one on each switch port. This will enable communication between two switches.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-5.svg)

But how will B send data to C ?

For that we will also need a **gateway**. It is a method of communicating to the outside networks.

To see the existing routing table configurations, we use:

```bash
route
```

To add the another host C on B, we use:

```bash
ip route add 192.168.2.0/24 via 192.168.1.1
```

> This has to be done for all the target hosts.

To connect the host to the internet `172.217.194.0/24`, we need to connect router to the internet. Then add a new routing configuration to the host, 

```bash
ip route add 172.217.194.0/24 via 192.168.2.1
```

There are so many sites for routing available on the internet. Instead of adding multiple routing table entries on same name, we can use :

```bash
$ ip route add default via 192.168.2.1
# or
$ ip route add 0.0.0.0 via 192.168.2.1
```

Now, any request outside your current network, goes through the router.

But this is OK, when you just have one router in your network, when you have multiple routers, you have to have multiple routing configurations.

**Next scenario**: How do you want to connect A to C ?

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-scenario-2.svg)

From A, if you ping IP<sub>C</sub>;

```bash
ping 192.168.2.5
# connect: network is unreachable
```

So in A, we need to have routing configurations in routing table,

```bash
ip route add 192.168.2.0/24 via 192.168.1.6
```

and in C, we need to have routing configurations in routing table,

```bash
ip route add 192.168.1.0/24 via 192.168.2.6
```

Now, we can ping IP<sub>C</sub> from A, but don't get any response from C, reason being by-default the Linux does not allow data packet forwarding. To enable that, in host C, overwrite value in `/proc/sys/net/ipv4/ip_forward`  file to 1 from 0.

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
1
cat /proc/sys/net/ipv4/ip_forward
1
```

> This change won't persist in across reboots, so it requires modification in `/etc/sysctl.conf` file.
>
> ```bash
> ...
> net.ipv4.ip_forward = 1
> ...
> ```

## DNS

![](https://raw.githubusercontent.com/aditya109/learning-k8s/a5821e04abcc0d86569ad9b827b130a1df3dc7aa/assets/networking-dns.svg)

Consider this setup. What we want to do is instead of its IP, we want to ping host B from A using its given name `db`. For that,

```bash
cat >> /etc/hosts
192.168.1.11	db
```

Now, `/etc/hosts` is the source of truth for host A. Whatever it sees in the `/etc/hosts` file it thinks it is truth and starts sending requests accordingly. 

The problem here is the entries in `/etc/hosts` file might not always be true. For example, in host B, running `hostname` command reveals that the actual name of host B is `host-2`.

But, as host A is unaware of this fact, whatever comes requests for formed for `db` host, it re-routes all the requests to host B.

This is called Name Resolution.

This process is not scalable though. For example, let's we have a 100 servers, and at least 3 of servers got their IPs updated.

Starting off, we will need to manually update `/etc/hosts` file for all 100 of the servers, and post that if some host IPs got updated, we have to manually do it for all the servers. We this all of these entries are now moved to a special server called DNS server.

To add a DNS server to any host, all you need to do is add an entry to the `/etc/resolv.conf` file.

```bash
cat /etc/resolv.conf
nameserver 		192.168.1.100

ping db
```

Now, all IP-hostname entries get updated in the one single place only, along with any host-IP changes done. Hence, the need for `/etc/hosts` is removed, which can still be used for private purposes.

But, what if there are 2 similar entries in the DNS server and `/etc/hosts/` file.

```bash
# DNS server entries
139.183.121.188			web
49.236.74.206			db
189.197.163.121			nfs
6.184.138.211			db-1
4.10.176.191			nfs-3
237.149.65.17			db-7
172.159.243.60			web-19
56.119.78.108			test			👈
72.149.228.30			nfs-prod
27.48.22.70				sql
```

```bash
# /etc/hosts
192.168.1.115			test			👈
```

Here, first `/etc/hosts` are looked at and then at DNS entries.

Now, what if I want to ping `netflix.com` from the host A ?

A new entry for Google DNS can be added to the local DNS server.

```bash
# DNS server entries
139.183.121.188			web
49.236.74.206			db
189.197.163.121			nfs
6.184.138.211			db-1
4.10.176.191			nfs-3
237.149.65.17			db-7
172.159.243.60			web-19
56.119.78.108			test			👈
72.149.228.30			nfs-prod
27.48.22.70				sql

Forward All to 8.8.8.8
```



Let's say there is an org DNS connected to host-A which has the following records:

```bash
# DNS server(192.168.1.100) entries
192.168.1.10 		web.mycompany.com
192.168.1.11 		db.mycompany.com
192.168.1.12 		nfs.mycompany.com
192.168.1.13 		web-1.mycompany.com
192.168.1.14 		sql.mycompany.com
192.168.1.15 		hr.mycompany.com
```

Now, whenever I hit `web`, it should re-route the traffic to `web.mycompany.com`. 

For that, we need to add an entry to `/etc/resolv.conf`.

```bash
cat /etc/resolv.conf
nameserver			192.168.1.100
search				mycompany.com
```

The search entry here appends `mycompany.com` to all the search entries and then searches it in the DNS.

Additional, entries in `search` field would mean either of the accumulated entries can be searched in the DNS. For example,

```bash
cat /etc/resolv.conf
nameserver			192.168.1.100
search				mycompany.com prod.mycompany.com
```

The searchable entries would be:

1. web.mycompany.com
2. web.prod.mycompany.com

### Record Types

| A     | web-server      | 192.168.1.1                       |
| ----- | --------------- | --------------------------------- |
| AAAA  | web-server      | fe80::e496:dacf:ecc7:62e2%9       |
| CNAME | food.web-server | eat.web-server, hungry.web-server |

### nslookup

The above command can be used to ping another hosts for an individual host.

> The individual host entries in `/etc/hosts` file are not considered for name resolution here, only DNS is considered.

```bash
nslookup www.google.com
Server:  dsldevice.lan
Address:  192.168.1.1

Non-authoritative answer:
Name:    www.google.com
Addresses:  2404:6800:4002:821::2004
          142.250.194.100
```

### dig

The above command can also be used to ping another hosts for an individual host with a lot more info.

> The individual host entries in `/etc/hosts` file are not considered for name resolution here, only DNS is considered.

```bash
dig www.google.com
```

## CoreDNS

### Configuring a dedicated system as DNS (CoreDNS)

1. Download the binary using curl or wget. And extract it. You get the coreDNS executable.

   ```bash
   wget https://github.com/coredns/coredns/releases/download/v1.8.4/coredns_1.8.4_linux_arm64.tgz
   ```

   ```bash
   tar -xzvf coredns_1.8.4_linux_arm64.tgz
   ```
   
   ```bash
   ./coredns
   ```
   
2. Run the executable to start a DNS server. (default port 53)

3. We haven't specified any IP-hostname mappings for which you need to provide some configurations.

   1. For that, put all of the entries into the DNS servers `/etc/hosts/` file.

4. And then we configure CoreDNS to use that file. CoreDNS loads it’s configuration from a file named `Corefile`. 
   Here is a simple configuration that instructs CoreDNS to fetch the IP to hostname mappings from the file `/etc/hosts`. When DNS server is run, it now picks the IPs and names from the `/etc/hosts/` file on the server.

## Network Namespaces

When a container is created, we create a network namespace for it. Within its own network namespace, the container can have its own virtual routing interface, routing table and ARP table.

### Creating network namespaces

Let's create 2 network namespaces `red` and `blue`. For that we use,

```shell
ip netns add red
ip netns add blue
```

To view all the network namespaces present on the system,

```bash
ip netns
```

### Exec in network namespaces

To view all the interfaces,

```shell
ip link
```

To view interfaces in specific network interfaces, we use:

```shell
ip netns exec red ip link
# or
ip -n red link
```

> So here we prevented containers from pitching the host network interfaces.

To display the ARP table for a particular IP address. It also shows all the entries of the ARP cache or table. 

```shell
arp
```

To ARP entries in a specific network interface, we use :

```shell
ip netns exec red arp
```

To display routing table entries we use:

```shell
route
# for specific network
ip netns exec red route
```

Network namespaces have no network connectivity, by default. And they cannot see their underlying host network.

### Connecting network interfaces

In order to connect to network interfaces together, we need a virtual cable or **pipe**. (virtual cable with two interfaces)

First we create a pipe with end-interfaces called `veth-red` and `veth-blue`.

```shell
ip link add veth-red type veth peer name veth-blue
```

Now we need to attach pipe's end-interfaces to the respective network namespaces.

```shell
ip link set veth-red netns red
ip link set veth-blue netns blue
```

 Now we need to assign IP addresses to the respective **pipe-end-interface**-**network-namespace** pair.

```shell
ip -n red addr add 192.168.15.1 dev veth-red
ip -n blue addr add 192.168.15.2 dev veth-red
```

Now we need to bring the interface up, for which we use:

```shell
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

Now you can ping the IP<sub>blue</sub> from red namespace.

```shell
ip netns exec red ping 192.168.15.2
```

You can view corresponding entries in respective namespace ARP tables as well.

```shell
ip netns exec red arp
ip netns exec blue arp
```

> `arp` on hostname who show no similar entries at all, with respect to either `blue` or `red` namespace.

### Enabling multiple namespaces for inter-communication

![](https://raw.githubusercontent.com/aditya109/learning-k8s//main/assets/multi-interface-inter-connection.png)



For this, we create a virtual network and a virtual switch, while connecting all the namespaces to the virtual switch.

We first create a bridge using `Linux Bridge` called `v-net-0`. From the perspective of the host it is exactly similar to the interface.

```shell
ip link add v-net-0 type bridge
```

We can view it using `ip link` command.

By default it is DOWN, so we need to turn it UP. For this, we use:

```shell
ip link set dev v-net-0 up
```

> To delete a virtual cable, we use:
>
> ```shell
> ip -n red link del veth-red
> ```
>
> Point to note here is when you delete one pipe-end-interface, the other one gets deleted immediately.

We need a fresh pipes.

```shell
ip link add veth-red type veth peer name veth-red-br
ip link add veth-blue type veth peer name veth-blue-br
```

Now we attach namespaces to the new pipes along with the bridge.

```shell
ip link set veth-red netns red
ip link set veth-red-br master v-net-0

ip link set veth-blue netns blue
ip link set veth-blue-br master v-net-0
```

Now we attach IP addresses to the network namespaces.

```shell
ip -n red addr add 192.168.15.1 dev veth-red
ip -n blue addr add 192.168.15.1 dev veth-blue
```

Now, we bring the respective interfaces `up`.

```shell
ip -n red link set veth-red up
ip -n blue link set veth-blue up
```

### Enabling communication between host machine and bridge

![](https://github.com/aditya109/learning-k8s/blob/main/assets/host-interface-communication.jpg?raw=true)

For this, we attach IP address to the bridge `v-net-0`.

```shell
ip addr add 192.168.15.5/24 dev v-net-0
```

Ping `192.168.15.5` from hostname.

```shell
ping 192.168.15.5
```

### Enabling connection Linux Bridge to external LAN connection (192.168.1.0)

If we directly try to ping this external IP `192.168.1.3` (which connected to external router `192.168.1.0`)from `blue` namespace, let's see what happens.

```shell
# first it checks in the routing table for existing connection
ip netns exec blue route

# as there would be no relevant entries present for such IP
# directly pinging external IP 192.168.1.3
>ip netns exec blue ping 192.168.1.3
Connect: Network is unreachable
```







```shell

```

```shell

```







```shell

```

```shell

```







```shell

```















## Docker Networking

## CNI

## Cluster Networking

## Pod Networking

## CNI in Kubernetes

## CNI Weave

## IP Address Management - Weave

## Service Networking

## DNS in Kubernetes

## Core DNS in Kubernetes

## Ingress

## Ingress - Annotations and rewrite-target

