# Security

## Contents

- [Kubernetes Security Primitives](#kubernetes-security-primitives)
  * [How to secure clusters?](#how-to-secure-clusters-)
    + [Step 1 : Defining who can access ?](#step-1---defining-who-can-access--)
    + [Step 2 : Once access is granted, defining what can they do ?](#step-2---once-access-is-granted--defining-what-can-they-do--)
- [Authentication](#authentication)
  * [How does the auth-mechanisms work for kube-apiserver?](#how-does-the-auth-mechanisms-work-for-kube-apiserver-)
    + [Static Password Files](#static-password-files)
      - [Auth Mechanisms - Basic](#auth-mechanisms---basic)
        * [Roles and role bindings for these users](#roles-and-role-bindings-for-these-users)
        * [To request for authentication](#to-request-for-authentication)
    + [Static Token Files](#static-token-files)
      - [Auth Mechanism - Basic](#auth-mechanism---basic)
        * [To request for authentication](#to-request-for-authentication-1)
- [A note on Service Accounts](#a-note-on-service-accounts)
  * [Service Account Controllers](#service-account-controllers)
  * [Default Service Account in a Pod and privilege](#default-service-account-in-a-pod-and-privilege)
  * [Custom service accounts](#custom-service-accounts)
- [TLS](#tls)
  * [TLS Basics](#tls-basics)
    + [Symmetric Encryption](#symmetric-encryption)
    + [Asymmetric Encryption](#asymmetric-encryption)
      - [Using Asymmetric Encryption to access cloud servers securely](#using-asymmetric-encryption-to-access-cloud-servers-securely)
  * [Accessing servers (bank)](#accessing-secure-servers--bank-)
  * [TLS in Kubernetes](#tls-in-kubernetes)
- [View Certificate Details](#view-certificate-details)
- [Certificates API](#certificates-api)
- [kube-config](#kube-config)
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

#### Symmetric Encryption

![](https://github.com/aditya109/learning-k8s/blob/main/assets/symmetric_enc.png?raw=true)

Here both the ends, sender and receiver use the same encryption keys for encoding and decoding which is shared over the same medium(which is considered *compromised*), meaning any sniffer in between will receive the cipher text as well as the key. This makes the encryption very insecure.

#### Asymmetric Encryption

![](https://github.com/aditya109/learning-k8s/blob/main/assets/asymmetric_enc.png?raw=true)

Here, there are 2 keys - private key and public key.

The *public key* can encode the text, which can only be decoded by the *private key*. 

##### Using Asymmetric Encryption to access cloud servers securely

Let assume there are 2 servers, `Server-1` and `Server-2`.

For any user `root`/`admin`, in order to access the servers, the user has to first create private and public keys.

```bash
> ssh-keygen
id_rsa id_rsa.pub
```

The user then has to place the public key into `~\.ssh\authorized_keys` directory.

Now in order to log into the server, the user has to pass the location of private key present in his computer.

```bash
ssh -i id_rsa admin@server1
Successfully logged in !
```

If another user wants to access the same servers, the `admin` user can place the their public keys in the same location `~\.ssh\authorized_keys` location, now the user can access the server in a similar manner.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/asymmetric_enc-asym_enc.svg?raw=true)

### Accessing secure servers (bank)

As mentioned in [Symmetric Encryption](#Symmetric Encryption), this method alone would not have worked to secure the servers.

In order to secure servers, we would have to use a combination of symmetric and asymmetric encyption.

Ok, let's roll back. Now taking the same example and assuming the communication channel is compromised.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/asymmetric_enc-bank-server-1.svg?raw=true)

First, the browser encrypts the user's credentials using some symmetric key and sends it over the compromised line. The sniffer now has the encrypted credentials but can do nothing as they do not have the symmetric key.

Then, the bank server generates their own public and private keys.

```bash
> openssl genrsa -out my-bank.key 1024
my-bank.key # public bank key
> openssl rsa -in my-bank.key -pubout > mybank.pem
my-bank.key mybank.pem # later one is private bank key
```

![](https://github.com/aditya109/learning-k8s/blob/main/assets/bank-server-access-2.png?raw=true)

Then, the bank sends the public key over the channel to the user, assuming that sniffer has already taken the copy of bank server public key.

The browser of the host then takes its generated symmetric key and encrypts it using the bank public key.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/bank-server-access-3.png?raw=true)

Now this encrypted chunk is sent over the channel, again assuming that sniffer has already taken the copy of encrypted chunk, but can do nothing as they only have bank server public key, and not their private key.

The bank receives the encrypted chunk and decrypts it using the bank server private key and gets the host's symmetric key.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/bank-server-access-4.png?raw=true)

But, the FME understands this trick. He comes up with an exact replica of the bank's website, and presents it to the user asking them to type out their credentials. He wants the user to think that the website is legit so he generates his own public and private key-pair and configures it on his own server, also somehow manages to tweak the user's environment to re-route requests going from the actual bank servers to his own servers.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/certificate.png?raw=true)

But we can actually identify this sniffer. The key which the any server sends is contained within a certificate. It is an actual certificate but in digital form. Naturally, the certificates are signed but that is how you can validate or in-validate the server.

> **CERTIFICATES**
>
> The certificate contains all the registered alias for the website, server location and several other information, etc. Above all, the certificate generated a FEM entity would be self-signed, and not signed by a certified authority. 
>
> In fact, the browser does this does for us, where it directly indicates `Not secure` in the address bar, to warn us about the insecurity of the site.
>
> CERTIFICATE AUTHORITIES or CAs are well-known organization which can sign and validate your certificate. Popular ones are `Symmantec`,  `GlobalSign`, `DigiCert`.
>
> **Process of Certificate Generation**
>
> 1. First the generated public key is used to create something called as `Certificate Signing Request` (CSR).
>
>    ```bash
>    > openssl req -new \
>    	-key my-bank.key \
>    	-out my.bank.csr \
>    	-subj "/C=US/ST=CA/O=MyOrg, Inc./CN=mydomain.com"
>    my-bank.key my-bank.csr
>    ```
>
> 2. CA then validates information and once it checks out they sign the certificate and send it back to the entity sending CSR.

The sniffer's CSR would fail at validation step itself.

But how would the browser validate the CA signature in the certificate ?

The CAs each have their own public and private keys. The public keys of the CAs are already built into the browsers. We can actually see the certificates within Trusted Root Certification Authorities under our Browser's Settings. The CAs use their private keys to sign the certificate. The public keys of the respective CAs are then used by the browser to validate the certificate themselves.

But this process works for public websites, what about the privately hosted websites ?

Most of the CAs mentioned above have their `On-Prem` versions available for usage. We can deploy these CAs onto our on-prem  servers, which can then be used to validate and sign certificates. These private CAs will then create public and private keys and public keys of these CAs will be manually installed on the browsers of the associated employees.



### TLS in Kubernetes

## View Certificate Details

## Certificates API

## kube-config

## Persistent Key/Value Store

## API Groups

## Authorization

## Role Based Access Controls

## Cluster Roles and Role Bindings

## Image Security

## Security Contexts

## Network Policy

## Developing network policies
