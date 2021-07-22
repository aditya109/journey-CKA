# Security

## Contents

- [Kubernetes Security Primitives](#kubernetes-security-primitives)
- [Authentication](#authentication)
- [Setting up Basic Authentication](#setting-up-basic-authentication)
- [A note on Service Accounts](#a-note-on-service-accounts)
- [TLS](#tls)
  * [TLS Basics](#tls-basics)
  * [TLS in Kubernetes](#tls-in-kubernetes)
- [View Certificate Details](#view-certificate-details)
- [Certificates API](#certificates-api)
- [KubeConfig](#kubeconfig)
- [Persistent Key/Value Store](#persistent-key-value-store)
- [API Groups](#api-groups)
- [Authorization](#authorization)
- [Role Based Access Controls](#role-based-access-controls)
- [Cluster Roles and Role Bindings](#cluster-roles-and-role-bindings)
- [Image Security](#image-security)
- [Security Contexts](#security-contexts)
- [Network Policy](#network-policy)
- [Developing network policies](#developing-network-policies)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## Kubernetes Security Primitives

### How to secure clusters?

#### Step 1 : Defining who can access ?

- Files - Username and Passwords
- Files - Username and Tokens
- Certificates
- External Authentication providers - LDAP
- Service Accounts

#### Step 2 : Once access is granted, defining what can they do ?

- RBAC Authorization
- ABAC Authorization
- Node Authorization
- Webhook Mode

## Authentication

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/Authorization.svg)

We can leave *Application End Users* for now.

Focusing on the following users,

1. User Accounts ✖
   1. Admins ✖
   2. Developers ✖
2. Service Accounts ✔
   1. Bots/Third-Party Applications

> We can not `kubectl create user user1` in Kubernetes.

However, we can create `serviceaccount` in Kubernetes.

```bash
kubernetes create serviceaccount sa1

kubernetes get serviceaccount
```

It will be covered under [A note on Service Accounts](#a-note-on-service-accounts).

For user accounts, all the requests via `kubectl` or `curl https://kube-server-ip:6443/` goes through the `kube-apiserver`, which in turn authenticates the requests, before processing it.

### How does the auth-mechanisms work for kube-apiserver?

We can have:

1. Static Password Files
2. Static Token Files
3. Certificates
4. Identity Services (third-party auth) - LDAP, Kerberos, etc.

#### Static Password Files

##### Auth Mechanisms - Basic

Let's assume we have a file called `user-details.csv`.

The table structure looks something like this:

|             |       |       |        |
| ----------- | ----- | ----- | ------ |
| password123 | user1 | u0001 | group1 |
| password123 | user2 | u0002 | group2 |
| password123 | user3 | u0003 | group1 |

First column, is the password, followed by username, user ID and group name.

We can then pass this file to `kube-apiserver` as an option `--basic-auth-file=user-details.csv`.

Or you can modify the manifest yaml present at `/etc/kubernetes/manifests/kube-apiserver.yaml`.

```yaml
apiVersion: v1
kind: Pod
metadata:
	name: kube-apiserver
	namespace: kube-system
spec:
	containers:
	- command:
		- kube-apiserver
		- --authorization-mode=Node, RBAC
		- --advertise-address=172.17.0.107
		...
		...
		- --basic-auth-file=/tmp/users/user-details.csv
		image: k8s.gcr.io/kube-apiserver:v1.9.7
		name: kube-apiserver
```

The `kubeadm`  tool will automatically pick up the changes and restart the `kube-apiserver`.

###### Roles and role bindings for these users

```yaml
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
 
---
# This role binding allows "jane" to read pods in the "default" namespace.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1 # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```



###### To request for authentication

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods -u "user1:password123"
```

####  Static Token Files

##### Auth Mechanism - Basic

Similar, thing can be done for token file as well.

```bash
--token-auth-file=user-details.csv
```

|                               |        |       |        |
| ----------------------------- | ------ | ----- | ------ |
| asdaf32jkh1k4jh124jhg23j4hgjk | user10 | u0001 | group1 |
| asdaf32jkh1k4jh124jhg23j4hgjk | user11 | u0002 | group  |
| asdaf32jkh1k4jh124jhg23j4hgjk | user12 | u0003 | group1 |

###### To request for authentication

```bash
curl -v -k https://master-node-ip:6443/api/v1/pods --header "Authorization: Bearer asdaf32jkh1k4jh124jhg23j4hgjk"
```

> This is not a recommended authentication mechanism.
>
> Consider volume mount while providing the auth file in a kubeadm setup.
>
> Setup Role-based Authorization for the new users.

## A note on Service Accounts

[Kubernetes Service Account in detail | Service Account tutorial - YouTube](https://www.youtube.com/watch?v=zfiz-oOeyMg&ab_channel=VivekSingh)

## TLS 

### TLS Basics

### TLS in Kubernetes

## View Certificate Details

## Certificates API

## KubeConfig

## Persistent Key/Value Store

## API Groups

## Authorization

## Role Based Access Controls

## Cluster Roles and Role Bindings

## Image Security

## Security Contexts

## Network Policy

## Developing network policies



