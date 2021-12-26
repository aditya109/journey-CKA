# Design and Install a Kubernetes Cluster

## Contents

## Design a Kubernetes Cluster

Ask before starting:

- Purpose of the cluster
  - Education - `Minikube` or `kubeadm`/`GCP`/`AWS`
  - Development & Testing
    - Multi-node cluster with a single master and multiple workers
    - Setup using `kubeadm` tool or quick provision on `GCP`/`AWS`/`AKS`
  - Hosting Production Applications
    - High available multi-node cluster with multiple master nodes
    - `kubeadm`/`GCP`/`KOPS`/`AWS` or other supported platforms
- Cloud or On Prem
  - Use `kubeadm` for on-prem
  - GKE for GCP
  - AKS for Azure
  - Kops for AWS
- Storage
  - High Performance - SSD Backed storage
  - Multiple concurrent connections - Network based storage
  - Persistent shared volumes for shared access across multiple PODs
  - Label nodes with specific disk types
  - Use Node selectors to assign applications to nodes with specific disk types
- Workloads
  - How many ?
  - What kind ?
    - Web
    - Big Data/Analytics
  - Application Resource Requirements
    - CPU Intensive
    - Memory Intensive
  - Traffic
    - Heavy Traffic
    - Burst Traffic

## Choosing Kubernetes Infrastructure

For local installation:

- Minikube - deploy VMs, create single node cluster
- `kubeadm` - requires VMs to be ready, single/multi- node cluster

Two types of solutions:

- Turnkey solutions - (requires VM provisioning, VM configuration, scripts for cluster deployment, VM maintenance). E.g., Kubernetes on AWS using KOPS
- Hosted solutions - (managed solutions) E.g., GKE

## Configure High Availability

Have multiple master nodes to provide multiple redundant schedulers and other control plane components.

### How does it work ?

Let's say we have 2 master nodes. We have another Kubernetes object `kube-controller-manager` endpoint.

In order to have multiple master nodes, while defining `kube-controller-manager` we provide a few options.

```sh
kube-controller-manager --leader-elect true
						--leader-elect-lease-duration 15s
						--leader-elect-renew-deadline 10s
						--leader-elect-retry-period 2s
```

## ETCD in HA

### How does it maintain consistency ?

Let's assume we have 3 replicas of ETCD running, and we want to write some data, how would it ensure consistency ?

Not all replicas are responsible for writing data, 1 of the instances is chosen to be leader, others becoming followers. Only the leader processes the write and makes sure that other replicas has the same data written on them.

*The write is considered as completed once the replication of data on majority of  replicas has occurred.*

#### How is leader elected ?

ETCD leader election protocol is called RAFT. Random timers are initiated on all 3 replicas of ETCD.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/etcd-1.svg?raw=true)

Which ever instance times out first, send the request to the other two instances for assuming leader role.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/etcd-2.svg?raw=true)

When the requests are approved, the requesting instance assumes as the leader.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/etcd-3.svg?raw=true)

 Here, as we can see `R2` gets the leadership role. Also `R2` continuously sends notification to other nodes that it is continuing to be the leader.
If for some reason `R2` does not send the same, election process is repeated between the remaining replicas i.e., `R1` and `R3`.

#### How are writes propagated ?

But what if 1 of the 2 followers has failed to replicate data or the replica itself is down.

> *The write is considered as completed once the replication of data on majority of  replicas has occurred.*

This means including leader, we wrote successfully on 2 out of 3 nodes, meaning the write is considered successful.

When the failing replica comes online, the data is replicated on that as well.

```
Majority/Quorum = floor(N/2 + 1); N = total number of nodes in the cluster
```

| Instances N                                                  | Quorum Q | Fault Tolerance *f=N-Q* |
| ------------------------------------------------------------ | -------- | ----------------------- |
| 1                                                            | 1        | 0                       |
| 2                                                            | 2        | 0                       |
| [👉](https://emojipedia.org/backhand-index-pointing-right/) 3 | 2        | 1                       |
| 4                                                            | 3        | 1                       |
| [👉](https://emojipedia.org/backhand-index-pointing-right/) 5 | 3        | 2                       |
| 6                                                            | 4        | 2                       |
| [👉](https://emojipedia.org/backhand-index-pointing-right/) 7 | 4        | 3                       |
|                                                              |          |                         |

> Always select odd pair of cluster installation, as in case of network partition, fault tolerance always persists. 

[Back to Contents ⬆](#Contents)





















