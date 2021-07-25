# Cluster Maintenance

## Contents

- [OS Upgrades](#os-upgrades)
- [Kubernetes Software Versions](#kubernetes-software-versions)
- [Cluster Upgrade Process](#cluster-upgrade-process)
  * [The Process](#the-process)
    + [Upgrade Strategies](#upgrade-strategies)
    + [kubeadm - upgrade](#kubeadm---upgrade)
- [Backup and Restore Methodologies](#backup-and-restore-methodologies)
  * [Backup Candidates](#backup-candidates)
    + [Backing Resource Configuration](#backing-resource-configuration)
    + [Backing ETCD Cluster](#backing-etcd-cluster)
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
🐱‍🏍 kubectl get nodes
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

#### Upgrade Strategies

The cluster upgrade process starts with the following steps:

1. First the master is brought down and upgraded.
   ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/ClusterMaintantence-Cluster upgrade.svg)

2. Now, to upgrade the worker nodes, we have 5 strategies:
   Reference link here: [Kubernetes deployment strategies (container-solutions.com)](https://blog.container-solutions.com/kubernetes-deployment-strategies)
   
   1. **Recreate Strategy**- A deployment defined with a strategy of type Recreate will terminate all the running instances then recreate them with the newer version.
   
      ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/ClusterMaintantence-Cluster upgrade-recreate.svg)
   
      The problem with this approach is at any point of time we don't have any worker instance running, which leads to downtime.
   
      ```yaml
      spec:
        replicas: 3
        strategy:
          type: Recreate
      ```
   
   2. **Ramped - slow rollout** - A ramped deployment updates the pods in a rolling update fashion, a secondary `ReplicaSet` is created with the new version of the application, then the number of the old version is decreased and the new version is increased until the correct number of replicas is reached.
   
      ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/Ramped-slow%20rollout.svg)
   
      ```yaml
      spec:
      	replicas: 3
      	strategy:
      		type: RollingUpdate
      		rollingUpdate:
      			maxSurge: 2    # how many pods we can add at a time
                  maxUnavailable: 0	# maxUnavailable define how many pods can be unavailable during the rolling update
      ```
   
      | Pros                                                         | Cons                             |
      | ------------------------------------------------------------ | -------------------------------- |
      | version is slowly released across instances                  | rollout/rollback can take time   |
      | convenient for stateful applications that can handle rebalancing of the data | supporting multiple APIs is hard |
      |                                                              | no control over traffic          |
   
   3. **Blue/Green - best to avoid API versioning issues** - A blue/green deployment differs from a ramped deployment because the *green* version of the application is deployed alongside the *blue version*. After testing the new version meets the requirements, we update the Kubernetes service object that plays the role of load balancer to send traffic to the new version by replacing the version label in the selector field.
   
      ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/strategy-Canary.svg)
   
      In the following example we use two `ReplicaSets` side by side, version A with three replicas (75% of the traffic), version B with one replica (25% of the traffic).
      Truncated deployment manifest version A:
   
      ```yaml
      spec:
        replicas: 3
      ```
   
      Truncated deployment manifest version B, note that we only start one replica of the application:
   
      ```yaml
      spec:
        replicas: 1
      ```
   
      | Pros                                                 | Cons                                                         |
      | ---------------------------------------------------- | ------------------------------------------------------------ |
      | version released for a subset of users               | slow rollout                                                 |
      | convenient for error rate and performance monitoring | fine tuned traffic distribution can be expensive (99% A/ 1% B = 99 pod A, 1 pod B) |
   
      The procedure used above is Kubernetes native, we adjust the number of replicas managed by a `ReplicaSet` to distribute the traffic amongst the versions.
   
      If you are not confident about the impact that the release of a new feature might have on the stability of the platform, a canary release strategy is suggested.
   
   4. **A/B testing  - best for feature testing on a subset of users** - A/B testing is really a technique for making business decisions based on statistics, rather than a deployment strategy. However, it is related and can be implemented using a canary deployment.
      In addition to distributing traffic amongst versions based on weight, you can precisely target a given pool of users based on a few parameters (cookie, user agent, etc.,). This technique is widely used to test conversion of a given feature that converts the most.
      `Istio`, like other service meshes, provides a fine-grained way to subdivide service instances with dynamic request routing  based on weights and/or HTTP headers.
      ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/ClusterMaintantence_a_b%20testing%20strategy.svg)
      
      | Pros                                       | Cons                                                         |
      | ------------------------------------------ | ------------------------------------------------------------ |
      | requires intelligent load balancer         | hard to troubleshoot errors for a given session, distributed tracing becomes mandatory |
      | several versions run in parallel           | not straightforward, you need to setup additional tools      |
      | full control over the traffic distribution |                                                              |

#### kubeadm - upgrade

> We can only go one minor version at a time.

First open docs at this [Upgrading kubeadm clusters | Kubernetes](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/). This link contains the instructions for upgrading the cluster.

```bash
# find the OS you are working with
cat /etc/*release

# figure out the exact target version of kubeadm
apt update
apt-cache madison kubeadm

# upgrade the control plane node
sh controlplane
## drain the controlplane
kubectl drain controlplane --ignore-daemonsets
## upgrade kubeadm 
apt-mark unhold kubeadm && \
apt-get update && \
apt-get upgrade -y kubeadm=1.19.6-00 && \
apt-mark hold kubeadm
### verify the kubeadm version
kubeadm version
kubeadm upgrade plan
kubeadm upgrade apply v1.19.6
## upgrade kubelet
apt-get upgrade -y kubelet=1.19.6-00
systemctl restart kubelet
kubectl get nodes

# first move workload from the target worker node to the other non-targeted worker nodes
# drain the target worker node; this also cordons the node, marking it as unscheduleable
kubectl drain node-1 --ignore-daemonsets 

# upgrade worker nodes
sh node-1
# upgrade kubeadm and kubelet

apt-mark unhold kubeadm && \
apt-get update && \
apt-get upgrade -y kubeadm=1.19.6-00 && \
apt-mark hold kubeadm
apt-get upgrade -y kubelet=1.19.6-00
kubeadm upgrade node config --kubelet-version v1.19.6
systemctl daemon-reload &&\
srestart kubelet
# uncordon target worker node back
kubectl uncordon node-1

# upgrade other worker nodes in the similar way
```

> Alternative to `kubectl upgrade plan`,
>
> ```bash
> kubectl -n kube-system get cm kubeadm-config -oyaml
> ```
>

## Backup and Restore Methodologies

### Backup Candidates

- Resource Configurations
- ETCD Cluster 
- Persistent Volumes

#### Backing Resource Configuration

One way, is to manually create `yaml` specification files for all resources and saving it to some VCS platform.

The other way is to query `kube-apiserver` and get all the resource configurations.

```bash
kubectl get all --all-namespace -o yaml > all-deploy-services.yaml
```

> We can open-source solutions like `Velero`(ARK) for this purpose. 

#### Backing ETCD Cluster

The data directory used by `ETCD` cluster can be configured to be backed up by backup tool.

The second thing is we can use built-in `snapshot save` utility from `ETCD`.

```bash
ETCDCTL_API=3 etcdctl \
	snapshot save snapshot.db 
```

This creates a `snapshot.db` to the current directory.

We can also save status of the backup.

```bash
ETCDCTL_API=3 etcdctl \
	snapshot status snapshot.db 

+--------------+--------------+-------------+-------------+
|	  HASH	   | REVISION	  |  TOTAL KEYS	| TOTAL SIZE  |
+--------------+--------------+-------------+-------------+
|	  e63b3fc5 |    473353	  |    875  	| 	 4.1 MB  |
+--------------+--------------+-------------+-------------+
```

##### Steps to restore from snapshot.db

```bash
# create your snapshot
ETCDCTL_API=3 etcdctl \
	snapshot save snapshot.db
	--endpoints=https://127.0.0.1:2379 \ # this is the default as ETCD is running on master node and exposed on localhost 2379
	--cacert=/etc/etcd/ca.crt \	# verify certificates of TLS=enabled secure servers using this CA bundle
	--cert=/etc/etcd/etcd-server.crt \ # identify secure client using this TLS certificate file
	--key=/etc/etcd/etcd-server.key	# identify secure client using this TLS key file 

# stop kube-apiserver 
service kube-apiserver stop

# restore kube-apiserver from snapshot backup file
ETCDCTL_API=3 etcdctl \
	snapshot restore snapshot.db 
	--data-dir /var/lib/etcd-from-backup \

# change the location of the backup directory, under HostPath flag to /var/lib/etcd-from-backup
cd /etc/kubernetes/manifests
nano e

# restart the daemon
systemctl daemon-reload

# restart etcd service
service etcd restart

# start the kube-apiserver
service kube-apiserver start
```

## Working with ETCDCTL

`etcdctl` is a command-line tool for `etcd`.

To make use of `etcdctl` for tasks such as backup and restore, make sure you set the ETCDCTL_API to 3.

```bash
export ETCDCTL_API=3
```

> In case you are using Docker Desktop like me and can't seem to use `etcdctl` command line utility, go to *Settings* > *Kubernetes* > Turn on *Show system containers (advanced)*.
>
> Now, do a `docker ps | findstr "etcd"`.

| CONTAINER    | IMAGE                  | COMMAND                  | CREATED       | STATUS       | PORTS | NAMES                                                        |
| ------------ | ---------------------- | ------------------------ | ------------- | ------------ | ----- | ------------------------------------------------------------ |
| d8369de7e279 | 0369cf4303ff           | `etcd --advertise-cl???` | 4 minutes ago | Up 4 minutes |       | `k8s_etcd_etcd-docker-desktop_kube-system_9ee5c1dcba083ea85bbd86049f83a21a_22` |
| ab7141571545 | k8s.gcr.io/pause:3.4.1 | `"/pause"`               | 4 minutes ago | Up 4 minutes |       | `k8s_POD_etcd-docker-desktop_kube-system_9ee5c1dcba083ea85bbd86049f83a21a_22` |

> To see the name of the pod, do a `kubectl get pods -n kube-system | findstr "etcd"`
>
> ```powershell
> 🐱‍🏍 kubectl get pods -n kube-system | findstr "etcd"
> etcd-docker-desktop                      1/1     Running   22         31d
> ```
>
> To see the details of the pod,
>
> ```powershell
> 🐱‍🏍 kubectl describe pod etcd-docker-desktop -n kube-system 
> Name:                 etcd-docker-desktop
> Namespace:            kube-system
> Priority:             2000001000
> Priority Class Name:  system-node-critical
> Node:                 docker-desktop/192.168.65.4
> Start Time:           Sat, 17 Jul 2021 20:25:00 +0530
> Labels:               component=etcd
>                       tier=control-plane
> Annotations:          kubeadm.kubernetes.io/etcd.advertise-client-urls: https://192.168.65.4:2379
>                       kubernetes.io/config.hash: 9ee5c1dcba083ea85bbd86049f83a21a
>                       kubernetes.io/config.mirror: 9ee5c1dcba083ea85bbd86049f83a21a
>                       kubernetes.io/config.seen: 2021-06-20T09:23:08.375232500Z
>                       kubernetes.io/config.source: file
> Status:               Running
> IP:                   192.168.65.4
> IPs:
>   IP:           192.168.65.4
> Controlled By:  Node/docker-desktop
> Containers:
>   etcd:
>     Container ID:  docker://d8369de7e2792293b8b34dcaaa5e17e535299ba9e0c6caae628b36b99a753811
>     Image:         k8s.gcr.io/etcd:3.4.13-0
>     Image ID:      docker-pullable://k8s.gcr.io/etcd@sha256:4ad90a11b55313b182afc186b9876c8e891531b8db4c9bf1541953021618d0e2
>     Port:          <none>
>     Host Port:     <none>
>     Command:
>       etcd
>       --advertise-client-urls=https://192.168.65.4:2379
>       --cert-file=/run/config/pki/etcd/server.crt		👈
>       --client-cert-auth=true
>       --data-dir=/var/lib/etcd
>       --initial-advertise-peer-urls=https://192.168.65.4:2380
>       --initial-cluster=docker-desktop=https://192.168.65.4:2380
>       --key-file=/run/config/pki/etcd/server.key		👈
>       --listen-client-urls=https://127.0.0.1:2379,https://192.168.65.4:2379
>       --listen-metrics-urls=http://127.0.0.1:2381
>       --listen-peer-urls=https://192.168.65.4:2380
>       --name=docker-desktop
>       --peer-cert-file=/run/config/pki/etcd/peer.crt
>       --peer-client-cert-auth=true
>       --peer-key-file=/run/config/pki/etcd/peer.key
>       --peer-trusted-ca-file=/run/config/pki/etcd/ca.crt
>       --snapshot-count=10000
>       --trusted-ca-file=/run/config/pki/etcd/ca.crt		👈
>     State:          Running
>       Started:      Thu, 22 Jul 2021 04:45:25 +0530
>     Last State:     Terminated
>       Reason:       Error
>       Exit Code:    255
>       Started:      Wed, 21 Jul 2021 21:44:47 +0530
>       Finished:     Thu, 22 Jul 2021 04:45:21 +0530
>     Ready:          True
>     Restart Count:  22
>     Requests:
>       cpu:                100m
>       ephemeral-storage:  100Mi
>       memory:             100Mi
>     Liveness:             http-get http://127.0.0.1:2381/health delay=10s timeout=15s period=10s #success=1 #failure=8
>     Startup:              http-get http://127.0.0.1:2381/health delay=10s timeout=15s period=10s #success=1 #failure=24
>     Environment:          <none>
>     Mounts:
>       /run/config/pki/etcd from etcd-certs (rw)		👈
>       /var/lib/etcd from etcd-data (rw)				👈
> Conditions:
>   Type              Status
>   Initialized       True 
>   Ready             True 
>   ContainersReady   True 
>   PodScheduled      True 
> Volumes:
>   etcd-certs:
>     Type:          HostPath (bare host directory volume)
>     Path:          /run/config/pki/etcd			👈
>     HostPathType:  DirectoryOrCreate
>   etcd-data:
>     Type:          HostPath (bare host directory volume)
>     Path:          /var/lib/etcd				👈
>     HostPathType:  DirectoryOrCreate
> QoS Class:         Burstable
> Node-Selectors:    <none>
> Tolerations:       :NoExecute op=Exists
> Events:
>   Type    Reason          Age    From     Message
>   ----    ------          ----   ----     -------
>   Normal  SandboxChanged  19h    kubelet  Pod sandbox changed, it will be killed and re-created.
>   Normal  Pulled          19h    kubelet  Container image "k8s.gcr.io/etcd:3.4.13-0" already present on machine
>   Normal  Created         19h    kubelet  Created container etcd
>   Normal  Started         19h    kubelet  Started container etcd
>   Normal  SandboxChanged  7h15m  kubelet  Pod sandbox changed, it will be killed and re-created.
>   Normal  Pulled          7h15m  kubelet  Container image "k8s.gcr.io/etcd:3.4.13-0" already present on machine
>   Normal  Created         7h15m  kubelet  Created container etcd
>   Normal  Started         7h15m  kubelet  Started container etcd
>   Normal  SandboxChanged  14m    kubelet  Pod sandbox changed, it will be killed and re-created.
>   Normal  Pulled          14m    kubelet  Container image "k8s.gcr.io/etcd:3.4.13-0" already present on machine
>   Normal  Created         14m    kubelet  Created container etcd
>   Normal  Started         14m    kubelet  Started container etcd
> 
> ```
>
> The version of `etcd` is under Image field of the output.























