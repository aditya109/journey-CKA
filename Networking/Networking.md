# Networking

## Contents

## Switching and Routing

Let's assume that we have 2 systems A and B. How will they communicate to each other ??

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-1.svg)

We connect them to a **switch**.
![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-2.svg)

To connect each system to the switch, we need to create some sort of an interface. For which, we use the following commands on both the systems.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/networking-3.svg)

```bash
$ ip link # to view the switch interface name
$ ip addr add 192.168.1.10/24 dev eth0 # assign the system A to networking interface
$ ip addr add 192.168.1.11/24 dev eth0 # assign the system B to networking interface
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
$ route
```

To add the another host C on B, we use:

```bash
$ ip route add 192.168.2.0/24 via 192.168.1.1
```

> This has to be done for all the target hosts.

To connect the host to the internet `172.217.194.0/24`, we need to connect router to the internet. Then add a new routing configuration to the host, 

```bash
$ ip route add 172.217.194.0/24 via 192.168.2.1
```







## DNS

## CoreDNS

## Network Namespaces

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

