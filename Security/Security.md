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
    + [Accessing secure bank servers](#accessing-secure-bank-servers)
    + [Naming Conventions for the keys](#naming-conventions-for-the-keys)
  * [TLS in Kubernetes](#tls-in-kubernetes)
    + [Server Certificates for Servers](#server-certificates-for-servers)
    + [Client Certificates for Clients](#client-certificates-for-clients)
- [TLS in Kubernetes - Certificate Creation](#tls-in-kubernetes---certificate-creation)
  * [TLS Certificate Generation](#tls-certificate-generation)
- [View Certificate Details](#view-certificate-details)
  * [Performing Certificate Health Check](#performing-certificate-health-check)
- [Certificates API and Workflow](#certificates-api-and-workflow)
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

#### Accessing secure bank servers

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

Finally this is how the bank server authentication looks like from Client-To-Server perspective.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/bank-server-access-5.png?raw=true)

> *But how would client authentication look like from Server-To-Client perspective?*
>
> The client also generate CSR with their public key, which is sent to bank servers, ensuring that their request is legit.
>
> ![](https://github.com/aditya109/learning-k8s/blob/main/assets/bank-server-access-6.png?raw=true)

This entire infrastructure used for building authentication is known as **Public Key Instructure (PKI)**.

#### Naming Conventions for the keys

Public keys come with extension either with `*.crt` or `*.pem`. 
For example,

```bash
server.crt
server.pem
client.crt
client.pem
```

Private keys come with extension either with `*.key` or `*-key.pem`. For example,

```bash
server.key
server-key.pem
client.key
client-key.pem
```

### TLS in Kubernetes

#### Server Certificates for Servers

*The names may vary for cluster-setup to cluster-setup.*

| Component       | Certificate (Public Key) | Private Key    |      |
| --------------- | ------------------------ | -------------- | ---- |
| kube-api-server | apiserver.crt            | apiserver.key  |      |
| etcd-server     | etcdserver.crt           | etcdserver.key |      |
| kubelet-server  | kubelet.crt              | kubelet.key    |      |

#### Client Certificates for Clients

*The names may vary for cluster-setup to cluster-setup.*

| Component                                         | Certificate (Public Key)     | Private Key                  |
| ------------------------------------------------- | ---------------------------- | ---------------------------- |
| admin (for `kubectl` REST API)                    | admin.crt                    | admin.key                    |
| kube-scheduler  (for `kubectl` REST API)          | scheduler.crt                | scheduler.key                |
| kube-controller-manager  (for `kubectl` REST API) | controller-manager.crt       | controller-manager.key       |
| kube-proxy  (for `kubectl` REST API)              | kube-proxy.crt               | kube-proxy.key               |
| kube-api-server (for talking to etcd-server)      | apiserver-etcd-client.crt    | apiserver-etcd-client.key    |
| kube-api-server (for talking to kublet-server)    | apiserver-kubelet-client.crt | apiserver-kubelet-client.key |
| kubelet-server (as a client)                      | kubelet-client.crt           | kubelet-client.key           |

The CA can also be setup within our Kubernetes cluster, which also has its own `ca.crt` and `ca.key`.

## TLS in Kubernetes - Certificate Creation

To generate certificates,we can use the following tools:

- EASYRSA
- OPENSSL
- CFSSL

### TLS Certificate Generation

**Kubernetes CA**

Step 1: We will **generate a private key** for Kubernetes CA.

```bash
> openssl genrsa -out ca.key 2048
ca.key
```

Step 2: We **generate a CSR** for Kubernetes CA.

```bash
> openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr
ca.csr
```

Step 3: We sign certificates using `ca.key` generated in Step 1.

```bash
> openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
ca.crt
```

**Admin User**

Step 4: We will **generate a private key** for Admin User.

```bash
> openssl genrsa -out admin.key 2048
admin.key
```

Step 5: We **generate a CSR** for Kubernetes CA.

```bash
> openssl req -new \
				-key admin.key \
				-subj "/CN=kube-admin/O=system:masters" \ 
# the above O=system:masters specifies the group to which the client belongs to
				-out admin.csr
admin.csr
```

Step 6: We sign certificates using `ca.key` generated in Step 1.

```bash
> openssl x509 -req -in admin.csr -CA ca.key -CAkey ca.key -out admin.crt
admin.crt
```

> We follow the same process to generate all the client certificates.
>
> For kube-controlplane components, we have to prefix `system:` to the name in Step 5. For example, for `kube-proxy`.
>
> ```bash
> > openssl req -new \
> 				-key admin.key \
> 				-subj "/CN=system:kube-proxy/O=system:masters" \ 
> # the above O=system:masters specifies the group to which the client belongs to
> 				-out admin.csr
> admin.csr
> ```

But what to do with generate admin certificates ?

We can directly make a request to `kube-api-server` using `admin.key`, `admin.crt` and `ca.crt`.

```bash
curl https://kube-apiserver:6443/api/v1/pods \	
		--key admin.key \
		--cert admin.crt \
		--cacert ca.crt
```

We can also create `kube-config.yaml` within the cluster setup.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: ca.crt
    server: https://kube-apiserver:6443
  name: kubernetes
kind: Config
users:
- name: kubernetes-admin
  user:
    client-certificate: admin.crt
    client-key: admin.key
```

We will need to provide `ca.crt` within every component of Kubernetes cluster.

`etcd` clusters usually have their separate CA deployed for peer-to-peer validation which is provided in `etcd.yaml`.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/etcd.png?raw=true)

What about `kube-api-server` and its aliases ?

The commonly used aliases `kubernetes`, `kubernetes.default`, `kubernetes.default.svc` and `kubernetes.default.svc.cluster.local`. Also many refer to it with its IP address as well. All of which should be present in certificates.

What about the aliases ??

For that we create an configuration file for `openssl`.

```yaml
[req]
req_extensions = v3_req
[v3_req]
basicContraints = CA:FALSE
keyUsage = nonRepudiation
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1
IP.2 = 172.17.0.87
```



```bash
# generate the private key
> openssl genrsa -out apiserver.key 2048
apiserver.key

# generate the CSR
> openssl req -new \
			-key apiserver.key \
			-subj "/CN=kube-apiserver" \
			-out apiserver.csr
			--config openssl.cnf 👈
apiserver.csr

# generate the certificate
openssl x509 -req \
			-in apiserver.csr \
			-CA ca.crt \
			-CAkey ca.key \
            -out apiserver.crt
 apiserver.crt
```

The location of these artifacts are passed in the creation of `kube-api-server` creation.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/kuebapiserver.png?raw=true)

Now, for `kubelet` server certificates. The names passed onto CSR for the servers is same as respective nodes' names not `kubelet` and finally pass these onto `kubelet` config files.

```yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io./v1beta1
authentication:
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet-node01.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-node01.key"
```

**Client Certificates for `kubelet`**

These are used by the `kubelet` to authenticate the `kube-apiserver`. The certificates need to have the node name in the following format `system:node:node01` with group as `SYSTEM:NODES`.

## View Certificate Details

### Performing Certificate Health Check

1. First find out how was the cluster deployed.

   1. *the hard way* - `cat /etc/systemd/system/kube-apiserver.service`
   2. *using kubeadm* - `cat /etc.kubernetes/manifests/kube-apiserver.yaml`

2. Find the location of the required certificates in the cluster.

   | Component        |        Type        | Certificate Path | CN Name | ALT Name | Organization | Issuer | Expiration |
   | ---------------- | :----------------: | :--------------: | :-----: | :------: | ------------ | ------ | ---------- |
   | `kube-apiserver` |       Server       |                  |         |          |              |        |            |
   | `kube-apiserver` |       Server       |                  |         |          |              |        |            |
   | `kube-apiserver` |       Server       |                  |         |          |              |        |            |
   | `kube-apiserver` | Client (`kubelet`) |                  |         |          |              |        |            |
   | `kube-apiserver` | Client (`kubelet`) |                  |         |          |              |        |            |
   | `kube-apiserver` |  Client (`etcd`)   |                  |         |          |              |        |            |
   | `kube-apiserver` |  Client (`etcd`)   |                  |         |          |              |        |            |
   | `kube-apiserver` |  Client (`etcd`)   |                  |         |          |              |        |            |

```yaml
# kube-apiserver.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - /bin/sh
    - -c
    - exec /usr/local/bin/kube-apiserver --v=2
      --cloud-config=/etc/gce.conf
      --address=127.0.0.1
      --allow-privileged=true
      --cloud-provider=gce
      # 👇
      --client-ca-file=/etc/srv/kubernetes/pki/ca-certificates.crt 
      --etcd-servers=http://127.0.0.1:2379
      --etcd-servers-overrides=/events#http://127.0.0.1:4002
      --secure-port=443
      # 👇
      --tls-cert-file=/etc/srv/kubernetes/pki/apiserver.crt 
      # 👇
      --tls-private-key-file=/etc/srv/kubernetes/pki/apiserver.key 
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
      --requestheader-client-ca-file=/etc/srv/kubernetes/pki/aggr_ca.crt
      --requestheader-allowed-names=aggregator
      --requestheader-extra-headers-prefix=X-Remote-Extra-
      --requestheader-group-headers=X-Remote-Group
      --requestheader-username-headers=X-Remote-User
      --proxy-client-cert-file=/etc/srv/kubernetes/pki/proxy_client.crt
      --proxy-client-key-file=/etc/srv/kubernetes/pki/proxy_client.key
      --enable-aggregator-routing=true
      --tls-cert-file=/etc/srv/kubernetes/pki/apiserver.crt
      --tls-private-key-file=/etc/srv/kubernetes/pki/apiserver.key
      # 👇
      --kubelet-client-certificate=/etc/srv/kubernetes/pki/apiserver-client.crt  	   # 👇
      --kubelet-client-key=/etc/srv/kubernetes/pki/apiserver-client.key
      --service-account-key-file=/etc/srv/kubernetes/pki/serviceaccount.crt
      --token-auth-file=/etc/srv/kubernetes/known_tokens.csv
      --basic-auth-file=/etc/srv/kubernetes/basic_auth.csv
      --storage-backend=etcd3
      --storage-media-type=application/vnd.kubernetes.protobuf
      --etcd-compaction-interval=150s
      --target-ram-mb=180
      --service-cluster-ip-range=10.51.240.0/20
      --audit-policy-file=/etc/audit_policy.config
      --audit-webhook-mode=batch
      --audit-webhook-config-file=/etc/audit_webhook.config
      --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ExtendedResourceToleration,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota
      --runtime-config=api/all=true
      --advertise-address=33.3.3.3
      --ssh-user=zorp
      --ssh-keyfile=/etc/srv/sshproxy/.sshkeyfile
      --authentication-token-webhook-config-file=/etc/gcp_authn.config
      --authorization-webhook-config-file=/etc/gcp_authz.config
      --authorization-mode=Node,RBAC,Webhook
      --allow-privileged=true 1>>/var/log/kube-apiserver.log
      2>&1
    image: k8s.gcr.io/kube-apiserver:v1.9.7
    ... 
    ...
```

Let's decode the server certificate of `kube-apiserver` .

```bash
> openssl x509 -in /etc/srv/kubernetes/pki/apiserver.crt -text -noout
```

Note the following fields:

- `Certificate` > `Signature Algorithm` > `Validity` : `Not After`
- `Certificate` > `Signature Algorithm` > `Subject`
- `Certificate` > `Signature Algorithm` > `X509v3 extensions` > `X509v3 Subject Alternative Name`
- `Certificate` > `Signature Algorithm` > `Issuer`

**Inspect Service Logs**

```powershell
> journalctl -u etcd.service -l # for from scratch setup 
```

**View Logs** 

```powershell
> kubectl logs etcd-master # for `kubeadm` setup
```

> Sometimes, if the core components like `kube-apiserver`, `etcd-server` are down, the `kubectl` command won't work.
>
> Then, it might be required to delve one level deeper into the docker.
>
> ```powershell
> > docker ps -a # list all the containers
> > docker logs <pod-name>
> ```
>

## Certificates API and Workflow

Certificates API is used to sign certificates upon validation of CSRs from the CA servers. 
This is a built-in automated solution for generating certificates for users and provide privileges, which would have otherwise been done manually.

To sign a certificate, whenever a CSR is received, the following steps are followed:

1. A CSR is received.

   ```bash
   > openssl genrsa -out jane.key 2048
   jane.key
   > openssl req 
   			-new \
   			-key jane.key \
   			-subj "/CN=jane" \
   			-out jane.csr
   jane.csr			
   > cat jane.csr
   -----BEGIN CERTIFICATE REQUEST-----
   MIICVDCCATwCAQAwDzENMAsGA1UEAwwEamFuZTCCASIwDQYJKoZIhvcNAQEBBQAD
   ggEPADCCAQoCggEBAJybaAAaSUysnz6D0TGLs8p8Zmew9+2FH+A59pj1SSt5V8Mg
   XjENC7/BYpSLzEueKsFqiew7MntuUKuJaK1tHwNaUGhOzLgqcdTTenZxi7kPmTdK
   cbaxbz859d7v/T7I8BjE0EuL5VNkonwoqN69BNtMmJJOCzlvhFWDQLM/aNW1qER0
   6zeLmbJv9la5g8jixLpLYBeuAvPm6RBkO4ncmENbboGma9/XBM/xVgviUAAqHUJX
   AQOrhuaZSw0mKZByL/aOo85n0iBWWexB8GImFyIQOaamkUfAlnItT7k7l6odWglb
   hUB7HuidMFL5edf7dWFUqzSBzQYIfedT20NovE8CAwEAAaAAMA0GCSqGSIb3DQEB
   CwUAA4IBAQAdGIiJY8LQqtvST2VcZwQNCVb/Pblj9cGRhNSxAf3VYVLsmxe9k+S7
   qXB20kEeSpN9VO0PG68bYqs+oeygemgaes5OUKZQFXU8W+OeUoN8H5F0pAmVSEcK
   Ywzmd1oiMB85q8u99L6HTr1OQjgP1shT/MqzyAkN8CNn8CQACjiVs3Go0lvSX76V
   59DjTphnhisRAb1KWN5ctUL46bo+HNf3vsXFSNdACcSuJm7Wjf4s2GDcjno9EarQ
   VfAmnQxedRHif+bJ5YogkZq+FBq7Aa2P308GYlw7p8wDT2Xuy1K2nyj96de10cp/
   UnQdIXAMV0bvMseYZCVZrVC3CzicB2jA
   -----END CERTIFICATE REQUEST-----
   ```

2. Then a `CertificateSigningRequest` object is created, wherein the above `jane.csr` is placed in `base64` encoded format.

   ```bash
   > cat jane.csr | base64
   LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5N
   QXNHQTFVRUF3d0VhbUZ1WlRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0VQQURDQ0FRb0Nn
   Z0VCQUp5YmFBQWFTVXlzbno2RDBUR0xzOHA4Wm1ldzkrMkZIK0E1OXBqMVNTdDVWOE1nClhqRU5D
   Ny9CWXBTTHpFdWVLc0ZxaWV3N01udHVVS3VKYUsxdEh3TmFVR2hPekxncWNkVFRlblp4aTdrUG1U
   ZEsKY2JheGJ6ODU5ZDd2L1Q3SThCakUwRXVMNVZOa29ud29xTjY5Qk50TW1KSk9Demx2aEZXRFFM
   TS9hTlcxcUVSMAo2emVMbWJKdjlsYTVnOGppeExwTFlCZXVBdlBtNlJCa080bmNtRU5iYm9HbWE5
   L1hCTS94Vmd2aVVBQXFIVUpYCkFRT3JodWFaU3cwbUtaQnlML2FPbzg1bjBpQldXZXhCOEdJbUZ5
   SVFPYWFta1VmQWxuSXRUN2s3bDZvZFdnbGIKaFVCN0h1aWRNRkw1ZWRmN2RXRlVxelNCelFZSWZl
   ZFQyME5vdkU4Q0F3RUFBYUFBTUEwR0NTcUdTSWIzRFFFQgpDd1VBQTRJQkFRQWRHSWlKWThMUXF0
   dlNUMlZjWndRTkNWYi9QYmxqOWNHUmhOU3hBZjNWWVZMc214ZTlrK1M3CnFYQjIwa0VlU3BOOVZP
   MFBHNjhiWXFzK29leWdlbWdhZXM1T1VLWlFGWFU4VytPZVVvTjhINUYwcEFtVlNFY0sKWXd6bWQx
   b2lNQjg1cTh1OTlMNkhUcjFPUWpnUDFzaFQvTXF6eUFrTjhDTm44Q1FBQ2ppVnMzR28wbHZTWDc2
   Vgo1OURqVHBobmhpc1JBYjFLV041Y3RVTDQ2Ym8rSE5mM3ZzWEZTTmRBQ2NTdUptN1dqZjRzMkdE
   Y2pubzlFYXJRClZmQW1uUXhlZFJIaWYrYko1WW9na1pxK0ZCcTdBYTJQMzA4R1lsdzdwOHdEVDJY
   dXkxSzJueWo5NmRlMTBjcC8KVW5RZElYQU1WMGJ2TXNlWVpDVlpyVkMzQ3ppY0IyakEKLS0tLS1F
   TkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
   ```

   ```yaml
   # jane-key.csr
   apiVersion: certificates.k8s.io/v1
   kind: CertificateSigningRequest
   metadata:
     name: jane
   spec:
     groups:
     - system:authenticated
     usages:
     - digital signature
     - key encipherment
     - server auth
     request:
       LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0VhbUZ1WlRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0VQQURDQ0FRb0NnZ0VCQUp5YmFBQWFTVXlzbno2RDBUR0xzOHA4Wm1ldzkrMkZIK0E1OXBqMVNTdDVWOE1nClhqRU5DNy9CWXBTTHpFdWVLc0ZxaWV3N01udHVVS3VKYUsxdEh3TmFVR2hPekxncWNkVFRlblp4aTdrUG1UZEsKY2JheGJ6ODU5ZDd2L1Q3SThCakUwRXVMNVZOa29ud29xTjY5Qk50TW1KSk9Demx2aEZXRFFMTS9hTlcxcUVSMAo2emVMbWJKdjlsYTVnOGppeExwTFlCZXVBdlBtNlJCa080bmNtRU5iYm9HbWE5L1hCTS94Vmd2aVVBQXFIVUpYCkFRT3JodWFaU3cwbUtaQnlML2FPbzg1bjBpQldXZXhCOEdJbUZ5SVFPYWFta1VmQWxuSXRUN2s3bDZvZFdnbGIKaFVCN0h1aWRNRkw1ZWRmN2RXRlVxelNCelFZSWZlZFQyME5vdkU4Q0F3RUFBYUFBTUEwR0NTcUdTSWIzRFFFQgpDd1VBQTRJQkFRQWRHSWlKWThMUXF0dlNUMlZjWndRTkNWYi9QYmxqOWNHUmhOU3hBZjNWWVZMc214ZTlrK1M3CnFYQjIwa0VlU3BOOVZPMFBHNjhiWXFzK29leWdlbWdhZXM1T1VLWlFGWFU4VytPZVVvTjhINUYwcEFtVlNFY0sKWXd6bWQxb2lNQjg1cTh1OTlMNkhUcjFPUWpnUDFzaFQvTXF6eUFrTjhDTm44Q1FBQ2ppVnMzR28wbHZTWDc2Vgo1OURqVHBobmhpc1JBYjFLV041Y3RVTDQ2Ym8rSE5mM3ZzWEZTTmRBQ2NTdUptN1dqZjRzMkdEY2pubzlFYXJRClZmQW1uUXhlZFJIaWYrYko1WW9na1pxK0ZCcTdBYTJQMzA4R1lsdzdwOHdEVDJYdXkxSzJueWo5NmRlMTBjcC8KVW5RZElYQU1WMGJ2TXNlWVpDVlpyVkMzQ3ppY0IyakEKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
     signerName: kubernetes.io/kube-apiserver-client
   ```

   > All `CertificateSigningRequest`s can be viewed by everyone.

   ```bash
   > kubectl create -f jane-key.csr
   certificatesigningrequest.certificates.k8s.io/jane created
   
   > kubectl get csr
   ```

   | NAME | AGE  | SIGNERNAME                          | REQUESTOR          | CONDITION |
   | ---- | ---- | ----------------------------------- | ------------------ | --------- |
   | jane | 59s  | kubernetes.io/kube-apiserver-client | docker-for-desktop | Pending   |

3. The request can be reviewed and approved/denied.

   ```bash
   > kubectl certificate approve jane
   certificatesigningrequest.certificates.k8s.io/jane approved
   > kubectl certificate deny jane
   certificatesigningrequest.certificates.k8s.io/jane denied
   ```

4. The certificate is then shared back to the user.

   ```bash
   > kubectl get csr jane -o yaml
    kubectl get csr jane -o yaml
   apiVersion: certificates.k8s.io/v1
   kind: CertificateSigningRequest
   metadata:
     creationTimestamp: "2021-08-01T14:16:30Z"
     name: jane
     resourceVersion: "31198"
     uid: 466161d6-a21d-482d-b75f-fed4ecea5735
   spec:
     groups:
     - system:masters
     - system:authenticated
     request: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0VhbUZ1WlRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRApnZ0VQQURDQ0FRb0NnZ0VCQUp5YmFBQWFTVXlzbno2RDBUR0xzOHA4Wm1ldzkrMkZIK0E1OXBqMVNTdDVWOE1nClhqRU5DNy9CWXBTTHpFdWVLc0ZxaWV3N01udHVVS3VKYUsxdEh3TmFVR2hPekxncWNkVFRlblp4aTdrUG1UZEsKY2JheGJ6ODU5ZDd2L1Q3SThCakUwRXVMNVZOa29ud29xTjY5Qk50TW1KSk9Demx2aEZXRFFMTS9hTlcxcUVSMAo2emVMbWJKdjlsYTVnOGppeExwTFlCZXVBdlBtNlJCa080bmNtRU5iYm9HbWE5L1hCTS94Vmd2aVVBQXFIVUpYCkFRT3JodWFaU3cwbUtaQnlML2FPbzg1bjBpQldXZXhCOEdJbUZ5SVFPYWFta1VmQWxuSXRUN2s3bDZvZFdnbGIKaFVCN0h1aWRNRkw1ZWRmN2RXRlVxelNCelFZSWZlZFQyME5vdkU4Q0F3RUFBYUFBTUEwR0NTcUdTSWIzRFFFQgpDd1VBQTRJQkFRQWRHSWlKWThMUXF0dlNUMlZjWndRTkNWYi9QYmxqOWNHUmhOU3hBZjNWWVZMc214ZTlrK1M3CnFYQjIwa0VlU3BOOVZPMFBHNjhiWXFzK29leWdlbWdhZXM1T1VLWlFGWFU4VytPZVVvTjhINUYwcEFtVlNFY0sKWXd6bWQxb2lNQjg1cTh1OTlMNkhUcjFPUWpnUDFzaFQvTXF6eUFrTjhDTm44Q1FBQ2ppVnMzR28wbHZTWDc2Vgo1OURqVHBobmhpc1JBYjFLV041Y3RVTDQ2Ym8rSE5mM3ZzWEZTTmRBQ2NTdUptN1dqZjRzMkdEY2pubzlFYXJRClZmQW1uUXhlZFJIaWYrYko1WW9na1pxK0ZCcTdBYTJQMzA4R1lsdzdwOHdEVDJYdXkxSzJueWo5NmRlMTBjcC8KVW5RZElYQU1WMGJ2TXNlWVpDVlpyVkMzQ3ppY0IyakEKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg==
     signerName: kubernetes.io/kube-apiserver-client
     usages:
     - digital signature
     - key encipherment
     - server auth
     username: docker-for-desktop
   status:
     conditions:
     - lastTransitionTime: "2021-08-01T14:19:35Z"
       lastUpdateTime: "2021-08-01T14:19:36Z"
       message: This CSR was approved by kubectl certificate approve.
       reason: KubectlApprove
       status: "True"
       type: Approved
     - lastTransitionTime: "2021-08-01T14:19:35Z"
       lastUpdateTime: "2021-08-01T14:19:35Z"
       message: 'invalid usage for client certificate: server auth'
       reason: SignerValidationFailure
       status: "True"
       type: Failed
   ```

   The `request` field value here can be extracted off and decoded to `base64`, which can be shared to the respective by the user.

   ```bash
   >  echo LS0tLS1CRUdJTiBDRVJUSUZJQ0FURSBSRVFVRVNULS0tLS0KTUlJQ1ZEQ0NBVHdDQVFBd0R6RU5NQXNHQTFVRUF3d0V......hlZFJIaWYrYko1WW9na1pxK0ZCcTdBYTJQMzA4R1lsdzdwOHdEVDJYdXkxSzJueWo5NmRlMTBjcC8KVW5RZElYQU1WMGJ2TXNlWVpDVlpyVkMzQ3ppY0IyakEKLS0tLS1FTkQgQ0VSVElGSUNBVEUgUkVRVUVTVC0tLS0tCg== |base64 --decode
   -----BEGIN CERTIFICATE REQUEST-----
   MIICVDCCATwCAQAwDzENMAsGA1UEAwwEamFuZTCCASIwDQYJKoZIhvcNAQEBBQAD
   ggEPADCCAQoCggEBAJybaAAaSUysnz6D0TGLs8p8Zmew9+2FH+A59pj1SSt5V8Mg
   XjENC7/BYpSLzEueKsFqiew7MntuUKuJaK1tHwNaUGhOzLgqcdTTenZxi7kPmTdK
   cbaxbz859d7v/T7I8BjE0EuL5VNkonwoqN69BNtMmJJOCzlvhFWDQLM/aNW1qER0
   6zeLmbJv9la5g8jixLpLYBeuAvPm6RBkO4ncmENbboGma9/XBM/xVgviUAAqHUJX
   AQOrhuaZSw0mKZByL/aOo85n0iBWWexB8GImFyIQOaamkUfAlnItT7k7l6odWglb
   hUB7HuidMFL5edf7dWFUqzSBzQYIfedT20NovE8CAwEAAaAAMA0GCSqGSIb3DQEB
   CwUAA4IBAQAdGIiJY8LQqtvST2VcZwQNCVb/Pblj9cGRhNSxAf3VYVLsmxe9k+S7
   qXB20kEeSpN9VO0PG68bYqs+oeygemgaes5OUKZQFXU8W+OeUoN8H5F0pAmVSEcK
   Ywzmd1oiMB85q8u99L6HTr1OQjgP1shT/MqzyAkN8CNn8CQACjiVs3Go0lvSX76V
   59DjTphnhisRAb1KWN5ctUL46bo+HNf3vsXFSNdACcSuJm7Wjf4s2GDcjno9EarQ
   VfAmnQxedRHif+bJ5YogkZq+FBq7Aa2P308GYlw7p8wDT2Xuy1K2nyj96de10cp/
   UnQdIXAMV0bvMseYZCVZrVC3CzicB2jA
   -----END CERTIFICATE REQUEST-----
   ```

All the certificate signing operations are performed under `Controller Manager` - which has `CSR-APPROVING` and `CSR-SIGNING` which perform these tasks.

```bash
> cat /etc/kubernetes/manifests/kube-controller-manager.yaml
```

`kube-controller-manager` manifest file has 2 fields with `spec` section :

- `--cluster-signing-cert-file`
- `--cluster-signing-key-file`

## kube-config

A user can be make a call to `kube-apiserver` in the following ways:

1. Using `cURL` command -

   ```bash
   > curl https://my-kube-playground:6443/api/v1/pods \
   --key admin.key
   --cert admin.crt
   --cacert ca.crt
   
   {
   	"kind" : "PodList",
   	"apiVersion": "v1",
   	"metadata": {
   		"selfLink": "/api/v1/pods",
   	},
   	"items": []
   }
   ```

2. Using `kubectl` command - 

   ```bash
   > kubectl get pods \
   			--server my-kube-playground:6443
   			--client-key admin.key
   			--client-certificate admin.crt
   			--certificate-authority ca.crt
   No resources found.
   ```

3. Using `kube-config` file:

   ```yaml
   # $HOME/.kube/config
   ```

   The `kube-config` file has 3 sections:

   | Sections | Description                                                  | Examples                                 | Remark relating to above command                             |
   | -------- | ------------------------------------------------------------ | ---------------------------------------- | ------------------------------------------------------------ |
   | Clusters | They are various Kubernetes clusters                         | Development, Production, Staging, etc.   | --server                                                     |
   | Users    | They are various interacting users with multiple types of privileges | Admin, Dev user, Prod user, etc.         | --client-key admin.key,		<br />--client-certificate admin.crt,<br />--certificate-authority ca.crt |
   | Contexts | They create a symbolic relationship                          | Admin@Production, Prod user@Google, etc. | Cluster@User                                                 |

   ```yaml
   # kube-config.yaml
   apiVersion: v1
   kind: Config
   
   clusters:
   - name: my-kube-playground
     cluster:
       certificate-authority:
       server: https://my-kube-playground:6443
   
   contexts:
   - name: my-kube-admin@my-kube-playground
     context:
       cluster: my-kube-playground
       user: my-kube-admin
   
   users:
   - name: my-kube-admin
     user:
       client-certificate: admin.crt
       client-key: admin.key
   ```

   By default, `kubectl` utility looks for `$HOME/.kube/config` path.

   ```bash
   > kubectl get pods
   	--kubeconfig config
   ```
   
   ```yaml
   # kube-config.yaml
   apiVersion: v1
   kind: Config
   
   current-context: dev-user@google # specifies the current context
   
   clusters:
   - name: my-kube-playground
     # (values hidden)
   - name: development
     # (values hidden)
   - name: production
     # (values hidden)
   - name: my-kube-playground
     # (values hidden)
     
   contexts:
   - name: my-kube-admin@my-kube-playground
     # (values hidden)
   - name: dev-user@google
     # (values hidden)
   - name: prod-user@production
     # (values hidden)
   
   users:
   - name: my-kube-admin
     # (values hidden)
   - name: admin
     # (values hidden)
   - name: dev-user
     # (values hidden)  
   - name: prod-user
     # (values hidden)
   ```
   
   > `current-context` specifies the current context in usage.
   
   To view the `kube-config`, we use:
   
   ```bash
   > kubectl config view
   apiVersion: v1
   clusters:
   - cluster:
       certificate-authority-data: DATA+OMITTED
       server: https://kubernetes.docker.internal:6443
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
       client-certificate-data: REDACTED
       client-key-data: REDACTED
   ```
   
   To view a custom kube-config file, we use the same command:
   
   ```bash
   kubectl config view --kubeconfig=my-custom-config
   ```
   
   To change `current-context`, we use:
   
   ```bash
   kubectl config use-context prod-user@production
   ```
   
   ### Using contexts in Namespaces
   
   ```yaml
   # kube-config-04.yaml
   apiVersion: v1
   kind: Config
   
   clusters:
   - name: production
     cluster:
       certificate-authority: /etc/kubernetes/pki/ca.crt
       server: https://172.17.0.51:6443
   
   contexts:
   - name: admin@production
     context:
       cluster: production
       user: admin
       namespace: finance
   
   users:
   - name: admin
     user:
       client-certificate:  /etc/kubernetes/pki/users/admin.crt
       client-key: /etc/kubernetes/pki/users/admin.key
   ```
   
   ### Providing certificates' credentials in kube-configs
   
   ```yaml
   # kube-config-04.yaml
   apiVersion: v1
   kind: Config
   
   clusters:
   - name: production
     cluster:
       certificate-authority-data:
       # paste the base64 encoded the certificate data here 
       server: https://172.17.0.51:6443
   
   contexts:
   - name: admin@production
     context:
       cluster: production
       user: admin
       namespace: finance
   
   users:
   - name: admin
     user:
       client-certificate-data:  # paste the base64 encoded the certificate data here 
       client-key-data:  # paste the base64 encoded the key data here 
   ```
   
   

## Persistent Key/Value Store

## API Groups

The kubernetes API server can be queried via `cURL` command.

Example 1:

```bash
> curl https://kubernetes.docker.internal:6443/version
```

```json
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.2", GitCommit:"092fbfbf53427de67cac1e9fa54aaa09a28371d7", GitTreeState:"clean", BuildDate:"2021-06-16T12:59:11Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"windows/amd64"}
Server Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.1", GitCommit:"5e58841cce77d4bc13713ad2b91fa0d961e69192", GitTreeState:"clean", BuildDate:"2021-05-12T14:12:29Z", GoVersion:"go1.16.4", Compiler:"gc", Platform:"linux/amd64"}
```

Example 2:

```bash
> curl --location --request GET 'https://kubernetes.docker.internal:6443/api/v1/pods' \
--header 'Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Im1GLTlBNGg1VEJBQVF1bWQ4cTRQX2hiUGpHUTlMdVo4U3VtTUVhb0FVSm8ifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tbmQ1dmciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjkwNWQwZGQyLWYyNjgtNGYxNC05ZWM0LTEwY2JlYjE5Njc5OCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.XXWWArytFl8S5511VrGhTLp7r5MEKoowjFYl1jcuC2nCG7HTfDwNycjuJK2Bf4A6c-iwhkS1HxGtnVnUqeg2Lho6lCkfRuHJflkN_KYhTBh2FeE95gjZ7hpBkCLeK5A91hDJ7tVtyJe2QyLI58g2JFMvRMENIIsbr3W66bDrMmnLW1uxKLO3lDoC3YXYMJO1n9HREhcltqi_d3ljwF5JOK7XPtT7aFMUTzHUcsURN5u1bbeYJTir-fg8LaQ8DPFFxsRv1BvFzRalMgho8Gz1InOO1CPJWs9gRot7BuKQKsdCxVLiIsKn4K2Pv8z85SFI2FKSGzFvD_lNV71ZnV9YTA'
```

These APIs are categorized into 6 categories:

1. `/metrics`
2. `/healthz`
3. `/version`
4. `/api`  👈
5. `/apis` 👈
6. `/logs`

`/api` are the set of Core APIs which has all the resources within itself.

![](https://github.com/aditya109/learning-k8s/blob/main/assets/api%20distribution.png?raw=true)

`/apis` are the set of Named-group APIs are more organized.

















## Authorization

## Role Based Access Controls

## Cluster Roles and Role Bindings

## Image Security

## Security Contexts

## Network Policy

## Developing network policies

[Back to Contents ⬆](#Contents)

