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

## Container Storage Interface (CSI)

## Volumes

## Persistent Volumes

## Persistent Volume Claims

## Using PVCs in PODs

## Application Configuration

## Storage Class

[Back to Contents ⬆](#Contents)

