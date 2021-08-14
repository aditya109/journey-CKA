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



## Volume Driver Plugins in Docker

## Container Storage Interface (CSI)

## Volumes

## Persistent Volumes

## Persistent Volume Claims

## Using PVCs in PODs

## Application Configuration

## Storage Class

[Back to Contents ⬆](#Contents)
