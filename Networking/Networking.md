# Networking

## Contents

- [Switching and Routing](#switching-and-routing)
- [DNS](#dns)
  - [Record Types](#record-types)
  - [nslookup](#nslookup)
  - [dig](#dig)
- [CoreDNS](#coredns)
  - [Configuring a dedicated system as DNS (CoreDNS)](#configuring-a-dedicated-system-as-dns--coredns-)
- [Network Namespaces](#network-namespaces)
  - [Creating network namespaces](#creating-network-namespaces)
  - [Exec in network namespaces](#exec-in-network-namespaces)
  - [Connecting network interfaces](#connecting-network-interfaces)
  - [Enabling multiple namespaces for inter-communication](#enabling-multiple-namespaces-for-inter-communication)
  - [Enabling communication between host machine and bridge](#enabling-communication-between-host-machine-and-bridge)
  - [Enabling connection Linux Bridge to external LAN connection (192.168.1.0)](#enabling-connection-linux-bridge-to-external-lan-connection--19216810-)
  - [Connectivity from network namespaces to internet](#connectivity-from-network-namespaces-to-internet)
  - [Connectivity from internet to network namespaces](#connectivity-from-internet-to-network-namespaces)
- [Docker Networking](#docker-networking)
- [CNI](#cni)
  - [CNI Directives For Container Runtime](#cni-directives-for-container-runtime)
  - [CNI Directives for Network Plugins](#cni-directives-for-network-plugins)
- [Cluster Networking](#cluster-networking)
- [How to implement the Kubernetes networking model](#how-to-implement-the-kubernetes-networking-model)
- [Pod Networking](#pod-networking)
- [CNI in Kubernetes](#cni-in-kubernetes)
- [CNI Weaveworks](#cni-weaveworks)
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

Now, we can ping IP<sub>C</sub> from A, but don't get any response from C, reason being by-default the Linux does not allow data packet forwarding. To enable that, in host C, overwrite value in `/proc/sys/net/ipv4/ip_forward` file to 1 from 0.

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
192.168.1.11 db
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
nameserver   192.168.1.100

ping db
```

Now, all IP-hostname entries get updated in the one single place only, along with any host-IP changes done. Hence, the need for `/etc/hosts` is removed, which can still be used for private purposes.

But, what if there are 2 similar entries in the DNS server and `/etc/hosts/` file.

```bash
# DNS server entries
139.183.121.188   web
49.236.74.206   db
189.197.163.121   nfs
6.184.138.211   db-1
4.10.176.191   nfs-3
237.149.65.17   db-7
172.159.243.60   web-19
56.119.78.108   test   👈
72.149.228.30   nfs-prod
27.48.22.70    sql
```

```bash
# /etc/hosts
192.168.1.115   test   👈
```

Here, first `/etc/hosts` are looked at and then at DNS entries.

Now, what if I want to ping `netflix.com` from the host A ?

A new entry for Google DNS can be added to the local DNS server.

```bash
# DNS server entries
139.183.121.188   web
49.236.74.206   db
189.197.163.121   nfs
6.184.138.211   db-1
4.10.176.191   nfs-3
237.149.65.17   db-7
172.159.243.60   web-19
56.119.78.108   test   👈
72.149.228.30   nfs-prod
27.48.22.70    sql

Forward All to 8.8.8.8
```

Let's say there is an org DNS connected to host-A which has the following records:

```bash
# DNS server(192.168.1.100) entries
192.168.1.10   web.mycompany.com
192.168.1.11   db.mycompany.com
192.168.1.12   nfs.mycompany.com
192.168.1.13   web-1.mycompany.com
192.168.1.14   sql.mycompany.com
192.168.1.15   hr.mycompany.com
```

Now, whenever I hit `web`, it should re-route the traffic to `web.mycompany.com`.

For that, we need to add an entry to `/etc/resolv.conf`.

```bash
cat /etc/resolv.conf
nameserver   192.168.1.100
search    mycompany.com
```

The search entry here appends `mycompany.com` to all the search entries and then searches it in the DNS.

Additional, entries in `search` field would mean either of the accumulated entries can be searched in the DNS. For example,

```bash
cat /etc/resolv.conf
nameserver   192.168.1.100
search    mycompany.com prod.mycompany.com
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
ip netns exec blue ping 192.168.1.3
Connect: Network is unreachable
```

![](https://github.com/aditya109/learning-k8s/blob/main/assets/external-network-connection-to-namespace.jpg?raw=true)

**Solution:** We need to ping the external IP `192.168.1.3` via external router `192.168.1.0`.

```shell
ip netns exec blue ip router add 192.168.1.0/24 via 192.168.15.5

# now if we ping external IP 192.168.1.3
ip netns exec blue ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.

# but we can see we do not get a response back, as the external router doesn't have our host IP; for which, we need NAT enabled on our gateway here.
iptables -t nat -A POSTROUTING -s 192.168.15.0/24 -j MASQUERADE

# now when we ping the outside world
ip netns exec blue ping 192.168.1.3
PING 192.168.1.3 (192.168.1.3) 56(84) bytes of data.
64 bytes from 192.168.1.3: icmp_seq=1 ttl=63 time=0.587 ms
64 bytes from 192.168.1.3: icmp_seq=1 ttl=63 time=0.466 ms
```

### Connectivity from network namespaces to internet

```shell
ip netns exec blue ping 8.8.8.8
Connect: Network is unreachable

# the reason again here is routing tables entry is missing
ip netns exec blue route

# adding the default routing entry
ip netns exec blue ip route add default via 192.168.15.5

# pinging now would work
ip netns exec blue ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=0.587 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=0.466 ms
```

### Connectivity from internet to network namespaces

```shell
ping 192.168.15.2
Connect: Network is unreachable
```

One way, is to let them know that network namespace can be reached via host IP, but that would not be secure, as we would be giving away our routing setup.

Second way is to setup an IPTABLE routing configuration.

```shell
iptables -t nat -a PREROUTING --dport 80 --to-destination 192.168.15.2:80 -j DNAT
```

> **Notes:**
>
> 1. While testing the Network Namespaces, if you come across issues where you can't ping one namespace from the other, make sure you set the NETMASK while setting IP Address. i.e.: `192.168.1.10/24`
>
>    ```shell
>    ip -n red addr add 192.168.1.10/24 dev veth-red
>    ```
>
> 2. Another thing to check is Firewall D/IP Table rules. Either add rules to IP Tables to allow traffic from one namespace to another. Or disable IP Tables all together (Only in a learning environment).

## Docker Networking

Let's assume a single server, with Docker installed on it. It has an `eth0` interface with IP `192.168.1.10` which is connected to the local network.

The following network options are available to us while creating containers:

1. `docker run --network none nginx`
   Here the docker container is not attached to any network. There is no incoming and outgoing connection. This is complete _network isolation_.

2. `docker run --network host nginx`
   Here the docker container is attached to host network on port **80**, meaning a container running a web application on port 80 will be able to run on URI `https://192.168.1.10:80`.

   > Only one instance of such a container can be run, only one application can use a port at the same time.

3. `docker run nginx`
   Here the docker container is attached to a bridge network where each container get their own respective private IP.

   Docker creates this network as `bridge`, as that is the name visible on the output of `docker network ls`, but on the host, the network name created is `docker0`(which is visible in the output of `ip link`).

   So `docker0` and `bridge` are one and the same thing.

   The `docker0` is given an IP which can be viewed with `ip addr` command. The `ip link` also tells us that `docker0` interface is down by-default. Docker also creates a network namespace for it. You can view that using `ip netns`.

**How does Docker attach the container to the bridge network ?**

Whenever a container is created, Docker creates a network namespace for it. We can view the namespaces created by Docker using `ip netns` command (slightly modified version).
`ip netns exec "${container_id}" ip -s link show eth0`

![](https://github.com/aditya109/learning-k8s/blob/main/assets/docker-networking.png?raw=true)

Docker creates a virtual cable and attaches its one end to the bridge network of Docker and the other end is tied to the container itself.

Here, if we use the following command,

```bash
ip netns
b3165c10a92b
```

Then if you use `ip link`, you would see `vethbb1c343@if7` in the output.

Also, the other end of the virtual cable can be viewed by the same command `ip link`.

```bash
ip -n b3165c10a92b link
eth0@if8
```

The container gets assigned a separate IP which can be viewed by `ip addr`

```bash
ip -n b3165c10a92b addr
```

Same process is iterated multiple times.

On being connected to bridge network, the container if is hosting a web application, it can be accessed docker on port **80**, but not from the host.

For the doing that, we need to publish the ports.

`docker run --publish HOST_PORT:CONTAINER_PORT <container-name>`

**How to view IP rules of Docker ?**

```shell
iptables -nvL -t nat
Chain Docker (2 references)
```

| target | prot | opt | source   | destination |                                |
| ------ | ---- | --- | -------- | ----------- | ------------------------------ |
| RETURN | all  | --  | anywhere | anywhere    |                                |
| DNAT   | tcp  | --  | anywhere | anywhere    | tcp dpt:8080 to :172.17.0.2:80 |

## CNI

**_Container Runtime_**

1. Create Network Namespaces

**_Bridge_** `bridge add <cid> <namespaceid>`

2. Create Bridge Network/Interface
3. Create VETH Pairs (pipes)
4. Attach pipes to Namespaces
5. Attach pipes to Bridge
6. Assign IP addresses
7. Bring the interfaces up
8. Enable NAT -IP Masquerade

### CNI Directives For Container Runtime

- Container Runtime must create network namespace.
- Identify network the container must attach to.
- Container Runtime to invoke Network Plugin (bridge) when container is Added.
- Container Runtime to invoke Network Plugin (bridge) when container is Deleted.
- JSON format of the Network Configuration.

Examples, Kubernetes, rkt, Mesos, etc.

> Reason why Docker does not follow CNI ?
>
> It has its own networking interface standards, called Container Network Model (CNM).

### CNI Directives for Network Plugins

- Must support command line arguments ADD/DEL/CHECK
- Must support parameter container id, network ns, etc.
- Must manage IP Address assignment to PODs.
- Must return results in a specific format.

Examples, Bridge, VLAN, IPVLAN, MACVLAN, WINDOWS, DHCP, host-local.

Also, `weaveworks`, `flannel`, `cilium`, `vmwareNSX`, etc.

## Cluster Networking

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/cluster-networking.svg)

There are some pre requisites for the cluster:

1. Each node must have an IP, node-name, and MAC address.
2. Some ports must be open on the nodes as well.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/mandatory-port-openings-on-node.png)

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/mandatory-port-openings-on-master-node.png)

**Master node(s)-**

| Protocol | Direction | Port Range | Purpose                 | Used by              |
| -------- | --------- | ---------- | ----------------------- | -------------------- |
| TCP      | Inbound   | 6443\*     | Kubernetes API server   | All                  |
| TCP      | Inbound   | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
| TCP      | Inbound   | 10250      | Kubelet API             | Self, control plane  |
| TCP      | Inbound   | 10251      | kube-scheduler          | Self                 |
| TCP      | Inbound   | 10252      | kube-controller manager | Self                 |

**Worker node(s)-**

| Protocol | Direction | Port Range  | Purpose               | Used by             |
| -------- | --------- | ----------- | --------------------- | ------------------- |
| TCP      | Inbound   | 10250       | Kubernetes API        | Self, control plane |
| TCP      | Inbound   | 30000-32767 | NodePort Services\*\* | All                 |

**Handy Commands for debugging networking issues on cluster**

```shell
ip link
```

```shell
ip addr
```

```shell
ip addr add 192.168.1.10/24 dev eth0
```

```shell
ip route
```

```shell
ip route add 192.168.1.10/24 via 192.168.2.1
```

```shell
cat /proc/sys/net/ipv4/ip_forward
1
```

```shell
arp
```

```shell
netstat -plnt
```

**What is the network interface configured for cluster connectivity on the master node?**

```shell
kubectl get nodes -o wide | grep controlplane

controlplane   Ready    control-plane,master   9m47s   v1.20.0   10.27.140.6   <none>        Ubuntu 18.04.5 LTS   5.4.0-1052-gcp   docker://19.3.0

# copy internal ip from there

ip a | grep -B2 10.27.140.6
7180: eth0@if7181: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:1b:8c:06 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.27.140.6/24 brd 10.27.140.255 scope global eth0

# eth0 is the network interface
```

**What is the MAC address of node01 ?**

```shell
arp node01
Address                  HWtype  HWaddress           Flags Mask            Iface
10.27.140.8              ether   02:42:0a:1b:8c:07   C                     eth0
```

**What is the IP address of the Default Gateway?**

```shell
ip route show default
default via 172.17.0.1 dev eth1
```

**Notice that ETCD is listening on two ports. Which of these have more client connections established?**

```shell
netstat -anp | grep etcd | grep PORT_NUMBER | wc -l
```

## How to implement the Kubernetes networking model

[Creating Highly Available clusters with kubeadm | Kubernetes](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#steps-for-the-first-control-plane-node)

## Pod Networking

Requisites for Networking Model

- Every POD should have an IP Address.
- Every POD should be able to communicate with every other POD in the same node.
- Every POD should be able to communicate with every other POD in the same node on other nodes with out NAT.

Example,

Let's say we have 3 nodes:

1. 192.168.1.11
2. 192.168.1.12
3. 192.168.1.13

These 3 nodes are attached to a LAN at 192.168.1.0

Now on each node, we create a bridge network

```shell
ip link add v-net-0 type bridge
# on all 3 nodes and bring them up
ip link set dev v-net-0 up
```

We then assign separate subnetted IPs to all the 3 nodes.

```shell
# on node 1
ip addr add 10.244.1.1/24 dev v-net-0
# on node 2
ip addr add 10.244.2.1/24 dev v-net-0
# on node 3
ip addr add 10.244.3.1/24 dev v-net-0
```

Then we write and execute `net-script.sh` a script file to run post every pod creation on node 1.

```shell
# create veth pair
ip link add .....

# attach veth pair
ip link set .....
ip link set .....

# assign IP Address
ip -n <namespace> addr add .....
ip -n <namespace> route add .....

# bring up interface
ip -n <namespace> link set .....
```

The same is done for all the nodes, which establishes the communication between pods inside the nodes.

To esatablish communication between pods in node 1 and pods in node 2 is

```shell
podOnNode1$ ping 10.244.2.2 #ip of pod on node 2
Connect: Network is unreachable

node1$ ip route add 10.244.2.2 via 192.168.1.12
podOnNode1$ ping 10.244.2.2 #ip of pod on node 2
64 bytes from 8.8.8.8: icmp_seq==1 tt=63 time=0.587 ms
```

Now, the same is done for all the nodes' containers. This works fine in this simple setup, but would require a lot more configuration in large workload cluster.

It is usually advised to do the same on a router if the network has any. But the problem remains that the same set of instructions has to be run each time a container is created.

This is where CNI comes into picture, helping Kubernetes, guiding it to the protocol for how the script would look like.

```shell
# net-script.sh

#ADD)
# create veth pair
# attach veth pair
# assign IP Address
# bring up interface

# DEL)
# delete veth pair
```

So each time a container is created, the `kubelet` looks for the CNI configuration in the directory specified during the time of creation.

```sh
--cni-conf-dir=/etc/cni/net.d
```

Then, it looks for the script in the similarily specified directory.

```shell
--cni-bin-dir=/etc/cni/bin
```

It then executes the script with the `add` parameter alongwith container-name and namespace.

```shell
./net-script.sh add <container> <namespace>
```

## CNI in Kubernetes

As per CNI,

- Container Runtime must create network namespace.
- Identify network the container must attach to.
- Container Runtime to invoke Network Plugin (bridge) when container is ADDed.
- Containe Runtime to invoke Network Plugin (bridge) when container is DELeted.
- JSON format of the Network Configuration.

## CNI Weaveworks

Weavework run peer to peer setup, where in it places a `Weavework` daemon/service on every running node. The communication inside a node happens between `weavework` daemon and `pods`, but outside the node happens between `weavework` daemons.

To deploy `weavework` daemons on cluster, we run,

```shell
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

To view `weavework` pods, we use `kubectl get pods -n kube-system | grep weave`.

Inspect the kubelet service and identify the network plugin configured for Kubernetes.

```sh
ps -aux | grep kubelet | grep --color network-plugin=
```

To check CNI solution applied on the system.

```shell
ll /opt/cni/bin
```

What is the CNI plugin configured to be used on this kubernetes cluster?

```sh
ls /etc/cni/net.d/
```

> Points to keep in mind:
>
> If asked to deploy `weavework` on cluster, by default the range of IP addresses and the subnet used by `weavework` is `10.32.0.0/12` and often times it overlaps with the host system IP addresses. For example, on a particular host, the command `ip a | grep eth0` yields the following:
>
> ```sh
> $ ip a | grep eth0
> 12396: eth0@if12397: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
>     inet 10.40.56.3/24 brd 10.40.56.255 scope global eth0
> ```
>
> If we deploy a weave manifest file directly without changing the default IP addresses it will overlap with the host system IP addresses and as a result, it's weave pods will go into an `Error` or `CrashLoopBackOff`.
>
> If we will go more deeper and inspect the logs then we can clearly see the issue
>
> ```sh
> $ kubectl logs -n kube-system weave-net-6mckb -c weave
> Network 10.32.0.0/12 overlaps with existing route 10.40.56.0/24 on host
> ```
>
> So we need to change the default IP address by adding `&env.IPALLOC_RANGE=10.50.0.0/16` option at the end of the manifest file. It should be look like as follows
>
> ```sh
> kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.50.0.0/16"
> ```

## IP Address Management - Weave

CNI Plugin Responsibilities:

- Must support arguments ADD/DEL/CHECK
- Must support parameters container id, network ns, etc.
- **Must manage IP Address assignment to PODs.**
- Must return results in a specific format.

But how to manage or get IPs ?

In order to do that, we either use `DHCP` or `host-local`. This can also be changed in the configuration for IPAM plugin.

```shell
cat /etc/cni/net.d/net-script.conf
{
 ....
 "ipam": {
  "type": "host-local",
  "subnet": "10.244.0.0/16",
  "routes": [
   {
    "dst" : "0.0.0.0/0"
   }
  ]
 }
 ....
}
```

Let's see how `weaveworks` does this IP management. `weavework` by default, has IP range `10.32.0.0/12`, implying ranging from IP `10.32.0.1, 10.32.0.2, ........, 10.47.255.254` (~close to a million IPs).

<https://www.baeldung.com/cs/get-ip-range-from-subnet-mask>

What is the POD IP address range configured by weave?

```shell
ip addr show <network-bridge-name>
```

What is the default gateway configured on the PODs scheduled on node01?

_Try scheduling a pod on node01 and check ip route output_

```shell
kubectl run busybox --image=busybox --command sleep 1000 --dry-run=client -o yaml > pod.yaml
# open pod.yaml and add nodeName field under spec section.
kubectl create -f pod.yaml
kubectl exec busybox -- sh
ip r # gives you the default gateway
```

## Service Networking

![complete diagram](https://github.com/aditya109/learning-k8s/blob/main/assets/service-networking.svg?raw=true)

This is how the pods communicate to each other in a cluster setup.

But how does it work ?

Let's observe a 3-node cluster setup as above.

With each node, we would have an IP and we would be running `kubelet` - which acts as an over-watch to observe the changes happening in the cluster via `kube-apiserver`.

Every time a pod has to be created on the cluster, the `kubelet` would be creating the pod and invoking CNI plugin to configure networking configurations on the pod.
Similarly, each node also run `kube-proxy`, which watches changes in the cluster through `kube-apiserver`. Everytime a service is created, it is created as a cluster-wide virtual object, with an assigned IP within a pre-defined range specified under the flag `--service-cluster-ip-range` while `kube-apiserver` installation.

The range can also be found out using :

```sh
ps aux | grep kube-apiserver | grep --service-cluster-ip-range
root       16488 17.3  2.0 1105664 326124 ?      Ssl  20:28   0:10 kube-apiserver --advertise-address=192.168.49.2 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/var/lib/minikube/certs/ca.crt --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota --enable-bootstrap-token-auth=true --etcd-cafile=/var/lib/minikube/certs/etcd/ca.crt --etcd-certfile=/var/lib/minikube/certs/apiserver-etcd-client.crt --etcd-keyfile=/var/lib/minikube/certs/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/var/lib/minikube/certs/apiserver-kubelet-client.crt --kubelet-client-key=/var/lib/minikube/certs/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/var/lib/minikube/certs/front-proxy-client.crt --proxy-client-key-file=/var/lib/minikube/certs/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/var/lib/minikube/certs/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=8443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/var/lib/minikube/certs/sa.pub --service-account-signing-key-file=/var/lib/minikube/certs/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/var/lib/minikube/certs/apiserver.crt --tls-private-key-file=/var/lib/minikube/certs/apiserver.key
```

As you can see here, it is `10.96.0.0/12`.

What `kube-proxy` does is it gets these service-related IPs, it creates forwarding rules on each node to re-route traffic to required pods.

But how are these rules created ?

`kube-proxy` creates these rules by 3 methods:

1. `ipvs`
2. `iptables` (default)
3. `userspace`

> These can be configured by setting `--proxy-made` flag while installing `kube-proxy`.
>
> To find the type of proxy mode we can use:
>
> ```sh
> kubectl logs <kube-proxy-pod-name> -n kube-system | grep proxy | grep mode
> ```

To see the iptable entries, please use:

```shell
iptables -L -t nat | grep <service_name>
```

To get the range of IP addresses configured for PODs on this cluster

```sh
kubectl logs <weave-pod-name> weave -n kube-system | grep ipalloc-range
```

To get IP range configured for the services with the cluster

```shell
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep cluster-ip-range
```

## DNS in Kubernetes

Let's say we have a pod `web` having the IP `10.244.2.5`. We want to access it via `test` pod having IP `10.244.1.5`.

In order to do so, we create a service `web-services` which gets assigned `10.107.37.188` IP. As soon as this service is created, an entry in Kube DNS is made for this service. Now any pod can reach the service by its name or full name. The entry-table looks something like this:

| Hostname    | Namespace | Type | Root          | IP Address    |
| ----------- | --------- | ---- | ------------- | ------------- |
| web-service | apps      | svc  | cluster.local | 10.107.37.188 |
| 10-244-2-5  | default   | pod  | cluster.local | 10.244.2.5    |
| 10-244-1-5  | apps      | pod  | cluster.local | 10.244.1.5    |

If `test` and `web` are in same namespace, we can `curl http://web-service`.
If `test` and `web` are in different namespace (`web` in `apps` and `test` in `default`), we can `curl http://web-service.apps`.

Finally using fully qualified domain name, we can `curl http://web-service.apps.svc.cluster.local`.

> Records for pods are not created implicitly.
> But if enabled, records of pods will be created, though under **Hostname** pods have their dash-replaced IPs displayed instead of their names.

## Core DNS in Kubernetes

In a Kubernetes cluster, we run an executable `./Coredns` which takes its configuration from `/etc/coredns/Corefile`.

```sh
.:53 {
 errors
 health
 kubernetes cluster.local in-addr.arpa ip6.arpa {
  pods insecure
  upstream
  fallthrough in-addr.arpa ip6.arpa
 }
 prometheus :9153
 proxy . /etc/resolv.conf
 cache 30
 reload
}
```

This is passed a configMap object into CoreDNS pods.

Each time a pod or service is created, the CoreDNS creates an entry for the specified object.

CoreDNS also creates a service tethered to itself.

```sh
kubectl get service -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   14d
```

For the pods, they need key-value pair in`/etc/resolv.conf` for DNS resolution. The CoreDNS server `kube-dns` IP is placed as the value for nameserver.

```sh
cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
```

Also this file has another entry called `search`, which helps the pod resolve the service.

> For pods, you always have to use their fully qualified name.

This is taken care by the `kubelet`, as this value is also placed within `/var/lib/kubelet/config.yaml`.

```sh
cat /var/lib/kubelet/config.yaml
...
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
```

> From the `hr` pod `nslookup` the `mysql` service and redirect the output to a file `/root/CKA/nslookup.out`.
>
> ```sh
> kubectl exec -it hr -- nslookup mysql.payroll > /root/CKA/nslookup.out
> ```

## Ingress

Let's say we are deploying an application on the URL `www.my-online-store.com` . Let's say we dockerized the application, using whose image we create a POD `wear` within a deployment. Also we deploy a `MySQL` POD and to connect to the `MySQL` POD we create `mysql-service` of type **ClusterIP**. To make the application available to the outside world, we create another service connected to `wear-service` of type **NodePort** available on Port 38080.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-ingress-1.svg?raw=true)

The application will now be visible on `http://<node-ip>:38080`.

Let's say now we want to scale the application as our customer base is increasing, so we deploy 3 pods in our deployment.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-ingress-2.svg?raw=true)

Now, we don't want the users to type node-IP everytime. So I can configure my DNS to point to the IP of the nodes.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-ingress-3.svg?raw=true)

Also, we do not want the users to remember the port as well, so we will bring an additional layer of `proxy-server` running on port 80 to redirect its traffic to respective port 38080.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-ingress-5.svg?raw=true)

This work for on-prem infra setups, but if infra was public cloud like GCP, AWS, etc.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-GCP%20ingress.svg?raw=true)

Now, let's say your company business grows and you now have multiple services for consumer usage, for example,

1. `www.my-online-store.com/wear`
2. `www.my-online-store.com/watch`

So now we deploy another deployment with `video` pods, running on below a `video-service` LoadBalancer-type service, which is going to deploy another `gcp load-balancer-2` to redirect its traffic to  port 38282. 

To re-direct overall traffic we will have to run `another load-balancer`, to re-direct traffic to respective load-balancers for endpoints `/wear` and `/video`.

Also, we have to SSL authentication to provide secured `http` URL for our server.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-GCP%20ingress-2.svg?raw=true)

The problem here is each LoadBalancer service comes with a load-balancer which is very costly. Each time we do this, the cost increases and so does the complexity of our application.

So we use Ingress for these usage.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/networking-GCP%20ingress-.svg?raw=true)











## Ingress - Annotations and rewrite-target
