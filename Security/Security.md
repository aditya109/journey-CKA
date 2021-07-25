# Security

## Contents

- [Kubernetes Security Primitives](#kubernetes-security-primitives)
  - [How to secure clusters?](#how-to-secure-clusters-)
    - [Step 1 : Defining who can access ?](#step-1---defining-who-can-access--)
    - [Step 2 : Once access is granted, defining what can they do ?](#step-2---once-access-is-granted--defining-what-can-they-do--)
- [Authentication](#authentication)
  - [How does the auth-mechanisms work for kube-apiserver?](#how-does-the-auth-mechanisms-work-for-kube-apiserver-)
    - [Static Password Files](#static-password-files)
      - [Auth Mechanisms - Basic](#auth-mechanisms---basic)
        - [Roles and role bindings for these users](#roles-and-role-bindings-for-these-users)
        - [To request for authentication](#to-request-for-authentication)
    - [Static Token Files](#static-token-files)
      - [Auth Mechanism - Basic](#auth-mechanism---basic)
        - [To request for authentication](#to-request-for-authentication-1)
- [A note on Service Accounts](#a-note-on-service-accounts)
  - [Service Account Controllers](#service-account-controllers)
  - [Default Service Account in a Pod and privilege](#default-service-account-in-a-pod-and-privilege)
  - [Custom service accounts](#custom-service-accounts)
- [TLS](#tls)
  - [TLS Basics](#tls-basics)
  - [TLS in Kubernetes](#tls-in-kubernetes)
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

![authPic](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/Authorization.svg)

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

#### Static Token Files

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

Reference link: [Kubernetes Service Account in detail | Service Account tutorial - YouTube](https://www.youtube.com/watch?v=zfiz-oOeyMg&ab_channel=VivekSingh)

Let's say we have a Kubernetes cluster running, a number of users would have to access the cluster, right.

- Admins (human users)
- Developers (human users)
- Clients (human users)
- TPA (non-human users) - managed by service accounts

A service account is an entity which provides necessary privileges to the access Kubernetes cluster.

### Service Account Controllers

There are some controllers, called *service account controllers*, running as part of Kubernetes Controller Manager, which are responsible to create a `default` service account on every Kubernetes namespace.

### Default Service Account in a Pod and privilege

If we don't specify a service account for a particular resource, for example, a pod, running in a given namespace, the `default` service account is tethered to that resource. So if that service account doesn't have to another resource requested by our pod, it won't be able to access it.

```powershell
kidad in System32 ❯ kubectl create ns test
namespace/test created
kidad in System32 ❯ kubectl get serviceaccounts -n test
NAME      SECRETS   AGE
default   1         58s
```

Each service account has a secret tethered to it. We can see associated secret mounted on to it.

```bash
kidad in System32 ❯ kubectl describe serviceaccounts -n test default
Name:                default
Namespace:           test
Labels:              <none>
Annotations:         <none>
Image pull secrets:  <none>
Mountable secrets:   default-token-k868l
Tokens:              default-token-k868l
Events:              <none>

kidad in System32 ❯ kubectl get secrets -n test
NAME                  TYPE                                  DATA   AGE
default-token-k868l   kubernetes.io/service-account-token   3      11m

kidad in System32 ❯ kubectl get secrets -n test default-token-k868l -oyaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiB.....URS0tLS0tCg==
  namespace: dGVzdA==
  token: ZXlKaGJHY2lPaUpTVXp......yWUVpdw==
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: default
    kubernetes.io/service-account.uid: 47ffa0ea-ab9c-45a9-9ba2-41c405fdd909
  creationTimestamp: "2021-07-23T02:38:36Z"
  name: default-token-k868l
  namespace: test
  resourceVersion: "107035"
  uid: 1601f4f7-123e-46bb-b208-03e88c67f033
type: kubernetes.io/service-account-token
```

The `ca.crt` is the base64 encoded the CA certificate used by the kube-apiserver.

We can actually check that by the going to the `/etc/kubernetes/pki/ca.crt` and comparing it with base64 decoded version of `ca.crt` from given secret.

The `token` is actually an encrypted JWT token, which is passed to the `kube-apiserver`.

Let's try to directly access the `kube-apiserver`.

Open `~/.kube`. Look for `config` file and search for sub-field called `server`.

> For Docker Desktop, you can still `cd` into above directory directly.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTi....FURS0tLS0tCg==
   server: https://kubernetes.docker.internal:6443   👈
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDR.....JUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUF.....SBLRVktLS0tLQo=

```

On hitting the url [https://kubernetes.docker.internal:6443](https://kubernetes.docker.internal:6443/), we get **403 Forbidden** error.

```json
{
    "kind": "Status",
    "apiVersion": "v1",
    "metadata": {},
    "status": "Failure",
    "message": "forbidden: User \"system:anonymous\" cannot get path \"/api\"",
    "reason": "Forbidden",
    "details": {},
    "code": 403
}
```

But we can actually connect to this server manually. Pick the token from the serviceaccount yaml `kubectl get secrets -n test -oyaml`, and hit that url again putting the base64 decoded token in the header. (yaml file contains the original token) or directly put the token from secret description `kubectl describe secrets -n test`.

```bash
curl https://kubernetes.docker.internal:6443/api --insecure --header "Authorization: Bearer REPLACE_THIS_WITH_DECODED_TOKEN"
```

You should now get a **202 response**.

```json
{
    "kind": "APIVersions",
    "versions": [
        "v1"
    ],
    "serverAddressByClientCIDRs": [
        {
            "clientCIDR": "0.0.0.0/0",
            "serverAddress": "192.168.65.4:6443"
        }
    ]
}
```

Let's run a pod in `test` namespace.

```powershell
kubectl run nginx --image nginx -n test
```

Let's `exec` into the pod.

```bash
kubectl exec -it -n test nginx -- bash
root@nginx:cd /var/run/secrets/kubernetes.io/serviceaccount
root@nginx:/var/run/secrets/kubernetes.io/serviceaccount# ls -l
total 0
lrwxrwxrwx 1 root root 13 Jul 24 14:38 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Jul 24 14:38 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Jul 24 14:38 token -> ..data/token
root@nginx:/var/run/secrets/kubernetes.io/serviceaccount# cat ca.crt
-----BEGIN CERTIFICATE-----
MIIC5zCCAc+gAwI....9fPz07Y0OBj2D/X
-----END CERTIFICATE-----
```

So let's say our application wants to connect to `kube-apiserver`. It is going to create a Kubernetes client which by-default which look for `/var/run/secrets/kubernetes.io/serviceaccount` location in the pod.

From there it fetches the necessary authorization prerequisites and uses them to hit `kube-apiserver`.

Giving a small example, create a deployment using `bitnami/kubectl` image and inject a `sleep` command.

```bash
kubectl create deployment kctl-depl --image bitnami/kubectl --dry-run -oyaml > deployment.yaml
```

Edit the yaml file and inject the command.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: kctl-depl
  name: kctl-depl
  namespace: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kctl-depl
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: kctl-depl
    spec:
      containers:
        - image: bitnami/kubectl
          name: kubectl
          command: ['sleep', '1000000'] 👈
          resources: {}
status: {}
```

Create the deployment using this manifest.

```bash
kubectl create -f deployment.yaml
kubectl get pods -n test
NAME                         READY   STATUS    RESTARTS   AGE
kctl-depl-7495488c7b-c25mn   1/1     Running   0          6m7s
```

Let's exec onto this pod.

```powershell
kubectl exec -it -n test kctl-depl-7495488c7b-c25mn -- bash
I have no name!@kctl-depl-7495488c7b-c25mn:/$ kubectl api-resources # gets us all the resources which can be accessed by this particular resource.
```

| NAME                            | SHORTNAMES | APIVERSION                           | NAMESPACED | KIND                           |
| ------------------------------- | ---------- | ------------------------------------ | ---------- | ------------------------------ |
| bindings                        |            | v1                                   | TRUE       | Binding                        |
| componentstatuses               | cs         | v1                                   | FALSE      | ComponentStatus                |
| configmaps                      | cm         | v1                                   | TRUE       | ConfigMap                      |
| endpoints                       | ep         | v1                                   | TRUE       | Endpoints                      |
| events                          | ev         | v1                                   | TRUE       | Event                          |
| limitranges                     | limits     | v1                                   | TRUE       | LimitRange                     |
| namespaces                      | ns         | v1                                   | FALSE      | Namespace                      |
| nodes                           | no         | v1                                   | FALSE      | Node                           |
| persistentvolumeclaims          | pvc        | v1                                   | TRUE       | PersistentVolumeClaim          |
| persistentvolumes               | pv         | v1                                   | FALSE      | PersistentVolume               |
| pods                            | po         | v1                                   | TRUE       | Pod                            |
| podtemplates                    |            | v1                                   | TRUE       | PodTemplate                    |
| replicationcontrollers          | rc         | v1                                   | TRUE       | ReplicationController          |
| resourcequotas                  | quota      | v1                                   | TRUE       | ResourceQuota                  |
| secrets                         |            | v1                                   | TRUE       | Secret                         |
| serviceaccounts                 | sa         | v1                                   | TRUE       | ServiceAccount                 |
| services                        | svc        | v1                                   | TRUE       | Service                        |
| mutatingwebhookconfigurations   |            | admissionregistration.k8s.io/v1      | FALSE      | MutatingWebhookConfiguration   |
| validatingwebhookconfigurations |            | admissionregistration.k8s.io/v1      | FALSE      | ValidatingWebhookConfiguration |
| customresourcedefinitions       | crd, crds  | apiextensions.k8s.io/v1              | FALSE      | CustomResourceDefinition       |
| apiservices                     |            | apiregistration.k8s.io/v1            | FALSE      | APIService                     |
| controllerrevisions             |            | apps/v1                              | TRUE       | ControllerRevision             |
| daemonsets                      | ds         | apps/v1                              | TRUE       | DaemonSet                      |
| deployments                     | deploy     | apps/v1                              | TRUE       | Deployment                     |
| replicasets                     | rs         | apps/v1                              | TRUE       | ReplicaSet                     |
| statefulsets                    | sts        | apps/v1                              | TRUE       | StatefulSet                    |
| tokenreviews                    |            | authentication.k8s.io/v1             | FALSE      | TokenReview                    |
| localsubjectaccessreviews       |            | authorization.k8s.io/v1              | TRUE       | LocalSubjectAccessReview       |
| selfsubjectaccessreviews        |            | authorization.k8s.io/v1              | FALSE      | SelfSubjectAccessReview        |
| selfsubjectrulesreviews         |            | authorization.k8s.io/v1              | FALSE      | SelfSubjectRulesReview         |
| subjectaccessreviews            |            | authorization.k8s.io/v1              | FALSE      | SubjectAccessReview            |
| horizontalpodautoscalers        | hpa        | autoscaling/v1                       | TRUE       | HorizontalPodAutoscaler        |
| cronjobs                        | cj         | batch/v1                             | TRUE       | CronJob                        |
| jobs                            |            | batch/v1                             | TRUE       | Job                            |
| certificatesigningrequests      | csr        | certificates.k8s.io/v1               | FALSE      | CertificateSigningRequest      |
| leases                          |            | coordination.k8s.io/v1               | TRUE       | Lease                          |
| endpointslices                  |            | discovery.k8s.io/v1                  | TRUE       | EndpointSlice                  |
| events                          | ev         | events.k8s.io/v1                     | TRUE       | Event                          |
| ingresses                       | ing        | extensions/v1beta1                   | TRUE       | Ingress                        |
| flowschemas                     |            | flowcontrol.apiserver.k8s.io/v1beta1 | FALSE      | FlowSchema                     |
| prioritylevelconfigurations     |            | flowcontrol.apiserver.k8s.io/v1beta1 | FALSE      | PriorityLevelConfiguration     |
| ingressclasses                  |            | networking.k8s.io/v1                 | FALSE      | IngressClass                   |
| ingresses                       | ing        | networking.k8s.io/v1                 | TRUE       | Ingress                        |
| networkpolicies                 | netpol     | networking.k8s.io/v1                 | TRUE       | NetworkPolicy                  |
| runtimeclasses                  |            | node.k8s.io/v1                       | FALSE      | RuntimeClass                   |
| poddisruptionbudgets            | pdb        | policy/v1                            | TRUE       | PodDisruptionBudget            |
| podsecuritypolicies             | psp        | policy/v1beta1                       | FALSE      | PodSecurityPolicy              |
| clusterrolebindings             |            | rbac.authorization.k8s.io/v1         | FALSE      | ClusterRoleBinding             |
| clusterroles                    |            | rbac.authorization.k8s.io/v1         | FALSE      | ClusterRole                    |
| rolebindings                    |            | rbac.authorization.k8s.io/v1         | TRUE       | RoleBinding                    |
| roles                           |            | rbac.authorization.k8s.io/v1         | TRUE       | Role                           |
| priorityclasses                 | pc         | scheduling.k8s.io/v1                 | FALSE      | PriorityClass                  |
| csidrivers                      |            | storage.k8s.io/v1                    | FALSE      | CSIDriver                      |
| csinodes                        |            | storage.k8s.io/v1                    | FALSE      | CSINode                        |
| csistoragecapacities            |            | storage.k8s.io/v1beta1               | TRUE       | CSIStorageCapacity             |
| storageclasses                  | sc         | storage.k8s.io/v1                    | FALSE      | StorageClass                   |
| volumeattachments               |            | storage.k8s.io/v1                    | FALSE      | VolumeAttachment               |

We can also disable by automatic service account mounting with the yaml file itself.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    name: nginx
    namespace: test2
spec:
  automountServiceAccountToken: false
  containers:
    - name: nginx
      image: nginx
      resources:
        limits:
          memory: '128Mi'
          cpu: '500m'
```

 Here, we won't be able to see those directories `/var/run/secrets` within the pod.

### Custom service accounts

```powershell
kubectl create serviceaccount customsa
serviceaccount/customsa created

kubectl get secrets
NAME                   TYPE                                  DATA   AGE
customsa-token-ktv9z   kubernetes.io/service-account-token   3      22s 👈
default-token-rnpqs    kubernetes.io/service-account-token   3      25h
```

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
