# Cluster Maintenance

## Contents

## OS Upgrades

![](D:\Work\aditya109\learning-myself\learning-k8s\5. Cluster Maintenance\assets\ClusterMaintantence.svg)

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

## Backup and Restore Methodologies

## Working with ETCDCTL





























