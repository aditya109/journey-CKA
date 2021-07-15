# Cluster Maintenance

## Contents

- [OS Upgrades](#os-upgrades)
- [Kubernetes Software Versions](#kubernetes-software-versions)
- [Cluster Upgrade Process](#cluster-upgrade-process)
  * [The Process](#the-process)
- [Backup and Restore Methodologies](#backup-and-restore-methodologies)
- [Working with ETCDCTL](#working-with-etcdctl)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## OS Upgrades

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/ClusterMaintantence.svg)

For a cluster upgrade, if normally the node goes down for maintenance and comes back online within 5 minutes, nothing happens, but if not so the pods are terminated on the node. If the pods were part of a `ReplicaSet`, the time it takes for coming back online is **pod-eviction-timeout**.

```powershell
kube-controller-manager --pod-eviction-timeout=5m0s
```

So, if a node is brought online within 5 mins, the maintenance task can be done, but we have no way of knowing if the node will or will not come back online.

Therefore, we follow a safer way. 

1. We `cordon` node, meaning we mark the node as *unschedulable*. We purposefully `drain` the specified node, the pods are deleted from the drain-target node and re-created on the remaining eligible nodes.

   ```bash
   kubectl cordon node-1 && kubectl drain node-1 --ignore-daemonsets --force --delete-emptydir-data
   ```

2. We then perform upgrades.

3. We then `uncordon` the node.

   ```bash
   kubectl uncordon node-1
   ```

## Kubernetes Software Versions

Getting Kubernetes version installed on nodes,

```bash
> kubectl get nodes
NAME             STATUS   ROLES                  AGE     VERSION
docker-desktop   Ready    control-plane,master   3h16m   v1.21.1
```

The version `v1.21.1` is Kubernetes version installed on the master node.

`v1` is the major version.
`21` is the minor version-contains new features and functionalities.
`1` is the patch version-contains.

## Cluster Upgrade Process

Kubernetes has the following components:

1. `kube-apiserver`
2. `controller-manager`
3. `kube-scheduler`
4. `kubelet`
5. `kube-proxy`
6. `kubectl`

*Do all the components need to be at the same version at a time?*

All components (except for `kubectl`) can be at a version less than or equal to the `kube-apiserver`.

> version(`kube-apiserver`) >= version (all other components)

So, let's `kube-apiserver` is a version x, then `controller-manager` and `kube-scheduler` can be at maximum of x-1 (one version lower than `kube-apiserver`), and  `kubelet` and `kube-proxy` can be at the very least at x-2.

`kubectl` can be x+1, x or x-1.

### The Process

The cluster upgrade process starts with the following steps:

1. First the master is brought down and upgraded.
   ![](<https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/ClusterMaintantence-Cluster upgrade.svg>)

2. Now, to upgrade the worker nodes, we have 5 strategies:

   1. **Recreate Strategy**- A deployment defined with a strategy of type Recreate will terminate all the running instances then recreate them with the newer version.

      ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/ClusterMaintantence-Cluster upgrade-recreate.svg)

      The problem with this approach is at any point of time we don't have any worker instance running, which leads to downtime.

      ```yaml
      spec:
        replicas: 3
        strategy:
          type: Recreate
      ```

      [Kubernetes deployment strategies (container-solutions.com)](https://blog.container-solutions.com/kubernetes-deployment-strategies)

   2. ****

   3. 

   4. 

   5. 


## Backup and Restore Methodologies

## Working with ETCDCTL

There is no apparent reason for the change. There can be no issue with the problem. the power of sick le 



























