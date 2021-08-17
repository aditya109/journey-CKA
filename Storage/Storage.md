# Security

## Contents

## Storage in Docker

There are 2 types of storage used in Docker:

- Storage Driver
- Volume Driver

When you install Docker on a system, it creates the following folder structure:

```bash
├───aufs
├───containers
├───image
├───var
│   └───lib
│       └───docker
└───volumes
```

### Layered Architecture

![](https://github.com/aditya109/learning-k8s/blob/main/assets/storage-docker.png?raw=true)

So each time a container is created the Image layers stay the same, but a fresh container layer is created. When any container is destroyed, the corresponding container layer is destroyed along with the changes within it.

But what if we want to modify our source code ?

We can, but before we save the source code, the container creates its own version of source code, and uses that code for execution. This is Copy-On-Write mechanism.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/copy-on-storage.png?raw=true)

So if we delete the container, the container layer goes along with it. In order to persist our source code, we use volumes.

```powershell
$ docker volume create data_volume
├───aufs
├───containers
├───image
└───var
    ├───lib
    │   └───docker
    └───volumes
        └───data_volume
# now this volume can be mounted
$ docker run -v data_volume:/var/lib/mysql mysql

```

![](https://github.com/aditya109/learning-k8s/blob/main/assets/volume-mounting.png?raw=true)

This is volume mounting.

#### Volume Mounting

What if we want to mount another container to the same location. This is an example of volume mounting.

```powershell
$ docker run -v data_volume:/var/lib/mysql mysql
```

#### Bind Mounting

But if we want our container to use an existing location on our live system. This is bind mounting.

```powershell
$ docker run -v /data/mysql:/var/lib/mysql mysql
```

> The -v option is outdated. Now we use `--mount` for mounting/binding volumes.
>
> ```powershell
> $ docker run \
> 	--mount type=bind, \
> 	  		source=/data/mysql, \
> 	  		target=/var/lib.mysql \
> 	 mysql
> ```
>
> 

#### Storage drivers

They help manage storage on images and containers.

- AUFS
- ZFS
- BTRFS
- Device Mapper
- Overlay
- Overlay2

## Volume Driver Plugins in Docker

The volumes are handled by volume driver plugins, not storage drivers, ***Local*** being the default.

- Azure File Storage
- Convoy
- DigitalOcean Block Storage
- Flocker
- gce-docker
- GlusterFS
- NetApp
- RexTay
- Portworx
- VMware cSphere Storage

```powershell
$ docker run-it \
 	--name mysql \
 	--volume-driver rexray/ebs \
 	--mount src=ebd-vol, target=/var/lib/mysql \
 	mysql
```

## Container Storage Interface (CSI)

### Container Runtime Interface (CRI)

CRI or Container Runtime Interface is a standard which defines how an orchestration solution like Kubernetes interacts with different runtimes, Docker, rkt, cri-o, etc.

### Container Networking Interface (CNI)

CNI or Container Networking Interface is a standard which defines how Kubernetes interacts with any networking renders like weaveworks, flannel, cilium, etc.

### Container Storage Interface (CSI)

Similarly same for storage, we have Container Storage Interface.

CSI moving parts:

- `RPC`
- `CreateVolume`
- `DeleteVolume`
- `ControllerPublishVolume`

Some of the CSI principles:

1. SHOULD call to provision a new volume
2. SHOULD call to delete a volume
3. SHOULD call to place a workload that uses the volume onto a node
4. SHOULD provision a new volume on the storage
5. SHOULD decommission a volume
6. SHOULD make the volume available on a node

## Volumes

Let's say we want to create a pod and mount a volume which is mapped to our local host `/data` directory, where it saves the file created during runtime.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
  labels:
    name: random-number-generator
spec:
  containers:
  - name: alpine
    image: alpine
    command:
      - "/bin/sh"
      - "-c"
    args: 
      - "shuf -i 0-100 -n 1 >> /opt/number.out;"
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8383
    volumes:
    - name: data-volume
      hostPath:
        path: /data
        type: Directory
```

But what if there is a multi-node cluster and we have some storage solution, like AWS EBS, etc. We cannot use localhost paths then.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: random-number-generator
  labels:
    name: random-number-generator
spec:
  containers:
  - name: alpine
    image: alpine
    command:
      - "/bin/sh"
      - "-c"
    args: 
      - "shuf -i 0-100 -n 1 >> /opt/number.out;"
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8383
    volumes:
    - name: data-volume
      awsElasticBlockStore:
        volumeID: <volume-id>
        fsType: ext4
```

## Persistent Volumes

A persistent volume is a cluster wide pool of volumes configured to used by users deploying applications.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  capacity:
    storage: {2:<Size>}
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

`accessModes` defines how a volume should be mounted on the host. There are 3 modes available:

- `ReadOnlyMany`
- `ReadWriteOnce`
- `ReadWriteMany`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  hostPath:
    path: /tmp/data
```

But what if there is a multi-node cluster and we have some storage solution, like AWS EBS, etc. We cannot use localhost paths then.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mypv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  awsElasticBlockStore:
    volumeID: <volume-id>
    fsType: ext4
```

```bash
$ kubectl create -f pv.yaml
$ kubectl get pv
```

## Persistent Volume Claims

The admin creates PV and the users create PVC, if the access is approved, the corresponding PVCs are mapped to PVs. 

Kubernetes tries to find satisfy *sufficient capacity* condition to the pods' PVCs. Other than that, we have:

- *Access Modes*
- *Volume Modes*
- *Storage Class*
- *Selector*

However, if there are like multiple matches for a PVC, we can still use labels and selectors to bind PVs to pod.

There is 1:1 relationship between PVC and PV, so once bounded no other pod can use the remaining space on the PV.

Also, if there is no match available for a PVC, the PVC will continue to be in `Pending` state, until newer volumes are made available in the cluster. Once the newer volume is available, the PVC will automatically be bound to the PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  resources:
    requests:
      storage: 500Mi
  accessModes:
    - ReadWriteOnce
```

```bash
$ kubectl create -f pvc-definition.yaml
$ kubectl get pvc
```

We initially created 1 PV `mypv` of 1Gi, since there are no other PVs left which have 500Mi capacity, the PVC `myclaim` is mapped to  `mypv`. The PVC is then moved to `Bound` status.

To delete a PVC, 

```bash
$ kubectl delete pvc myclaim
```

But on default, once PVC is deleted, what happens to the PV.

By default, the property `persistentVolumeReclaimPolicy` of a PV is set to `Retain`, meaning the PV will remain till it is manually deleted by the admin, but it is not available to be bound to any other PVC either. 

We have multiple options for `persistentVolumeReclaimPolicy` property of the PVC:

- `Delete` - PV deletion follows PVC deletion, freeing up storage.
- `Recycle` - PV data wipe follows PVC deletion, PV is available to be bound to other PVCs. 

## Using PVCs in PODs

Once we create a PVC we use it in a POD definition file by specifying the PVC Claim name under `persistentVolumeClaim` section in the volumes section like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/var/www/html"
        name: mypd
  volumes:
    - name: mypd
      persistentVolumeClaim:
        claimName: myclaim
```

The same is true for ReplicaSets or Deployments. Add this to the pod template section of a Deployment on ReplicaSet.

## Application Configuration

## Storage Class

[Back to Contents ⬆](#Contents)

