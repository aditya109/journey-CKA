# Table of Contents

- [Core Concepts](#core-concepts)
  * [Kubernetes Architecture](#kubernetes-architecture)
  * [ETCD](#etcd)
      - [Install ETCD](#install-etcd)
      - [Operate ETCD](#operate-etcd)
      - [ETCD Cluster](#etcd-cluster)
  * [Kube-API Server](#kube-api-server)
  * [Controller Manager](#controller-manager)
      - [View kube-controller-manager – kubeadm](#view-kube-controller-manager---kubeadm)
  * [Kube Scheduler](#kube-scheduler)
      - [View kube-scheduler – kubeadm](#view-kube-scheduler---kubeadm)
  * [Kubelet](#kubelet)
  * [Kube Proxy](#kube-proxy)
    + [PODs](#pods)
      - [Using PODs](#using-pods)
      - [Pod Definition YAML](#pod-definition-yaml)
    + [ReplicaSets](#replicasets)
      - [Labels and Selectors](#labels-and-selectors)
      - [Scaling Replicas](#scaling-replicas)
    + [Deployments](#deployments)
    + [Namespaces](#namespaces)
      - [Resource Quota](#resource-quota)
    + [Services](#services)
      - [Service Types:](#service-types-)
      - [Services NodePort](#services-nodeport)
      - [Services ClusterIP](#services-clusterip)
      - [Services LoadBalancer](#services-loadbalancer)
    + [Kubernetes Imperative and Declarative](#kubernetes-imperative-and-declarative)
      - [Imperative Commands](#imperative-commands)
        * [Create objects](#create-objects)
        * [Update Objects](#update-objects)
      - [Declarative Commands](#declarative-commands)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

 

# Core Concepts

## Kubernetes Architecture

Two types of Nodes:

1. **Worker Nodes**: Host Application as Containers

   **a.   Container Runtime Engine**

   **b.   kubelet**

   **c.    kube-proxy**

2. **Master Nodes**: Manage, Plan, Schedule, Monitor Nodes

   **a.    ETCD Cluster**

   **b.   kube-scheduler**

   **c.    Controller-Managers**

   **d.    kube-apiserver**
   
   ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image1.svg?token=AFH4ROZZHDUCY5XOXJAPG7DAAF65C)

## ETCD

ETCD is a distributed reliable **key-value** store that is simple, secure & fast.

#### Install ETCD

1. Download Binaries

2. Extract it

3. Run ETCD Service (Port 2379)

#### Operate ETCD

`.etcdctl set key1 value1` This command sets an entry in ETCD Store.

`.etcdctl get key1` This command gets the entry under `key1` from ETCD Store.

`.etcdctl ` This command is the `.etcdctl` help command.

#### ETCD Cluster

It records every detail of Kubernetes entity like:

1. **Nodes**
2. **PODs**
3. **Configs**
4. **Secrets**
5. **Accounts**
6. **Roles**
7. **Bindings**
8. **Others**

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image2.svg?token=AFH4RO6L65NFJBDNUOUGQCLAAF7M6)

(Optional) Additional information about ``ETCDCTL`` Utility

 ``ETCDCTL`` is the CLI tool used to interact with ETCD.

 ``ETCDCTL`` can interact with ETCD Server using 2 API versions - Version 2 and Version 3. 
By default, its set to use Version 2. Each version has different sets of commands.

For example, ``ETCDCTL`` version 2 supports the following commands:

1.  `etcdctl backup`

2.  `etcdctl cluster-health`

3.  `etcdctl mk`

4.  `etcdctl mkdir`

5.  `etcdctl set`

Whereas the commands are different in version 3

1.  `etcdctl snapshot save` 

2.  `etcdctl endpoint health`

3.  `etcdctl get`

4.  `etcdctl put`


 To set the right version of API set the environment variable `ETCDCTL_API` command

```powershell
export ETCDCTL_API=3
```

When API version is not set, it is assumed to be set to version 2. And version 3 commands listed above don't work. When API version is set to version 3, version 2 commands listed above don't work.

Apart from that, you must also specify path to certificate files so that `ETCDCTL` can authenticate to the ETCD API Server. The certificate files are available in the etcd-master at the following path. We discuss more about certificates in the security section of this course. So don't worry if this looks complex:

1.  `--cacert /etc/kubernetes/pki/etcd/ca.crt`   

2.  `--cert /etc/kubernetes/pki/etcd/server.crt`   

3.  `--key /etc/kubernetes/pki/etcd/server.key`

So, for the commands I showed in the previous video to work you must specify the `ETCDCTL` API version and path to certificate files. Below is the final form:

## Kube-API Server

The API server is a component of the Kubernetes control plane that exposes the Kubernetes API. The API server is the front end for the Kubernetes control plane. kube-apiserver is designed to scale horizontally—that is, it scales by deploying more instances.

Only kube-apiserver talk to ETCD cluster, rest of the control-plane components talk to ETCD cluster via kube-apiserver.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image3.svg?token=AFH4RO5C25M7GM62MTXU2JDAAF7WA)

```powershell
Aditya :: learning-k8s » kubectl get pods -n kube-system
NAME                                     READY   STATUS    RESTARTS   AGE
coredns-f9fd979d6-blxfl                  1/1     Running   0          6m13s
coredns-f9fd979d6-v54dp                  1/1     Running   0          6m13s
etcd-docker-desktop                      1/1     Running   0          5m
kube-apiserver-docker-desktop            1/1     Running   0          5m
kube-controller-manager-docker-desktop   1/1     Running   0          5m5s
kube-proxy-zk49d                         1/1     Running   0          6m13s
kube-scheduler-docker-desktop            1/1     Running   0          5m11s
storage-provisioner                      1/1     Running   0          4m58s
vpnkit-controller                        1/1     Running   0          4m57s
```

This command gives us all control-plane components running in kube-system namespace.

To view the manifest of kube-apiserver(kubeadm setup): 
`cat /etc/kubernetes/manifests/kube-apiserver.yaml`

To view options of kube-apiserver(non-kubeadm setup): 

```powershell
cat /etc/system/system/kube-apiserver.service
ps -aux | grep kube-apiserver
```

## Controller Manager

In Kubernetes, controllers are control loops that watch the state of your cluster, then make or request changes where needed. Each controller tries to move the current cluster state closer to the desired state.

Logically, each controller is a separate process, but to reduce complexity, they are all compiled into a single binary and run in a single process.

These controllers include:

1.    **Node Controller**: Responsible for noticing and responding when nodes go down.

2.    **Replication controller**: Responsible for maintaining the correct number of pods for every replication controller object in the system.

3.    **Endpoints controller**: Populates the Endpoints object (that is, joins Services & Pods).

4.    **Service Account & Token controllers**: Create default accounts and API access tokens for new namespaces.

- Watch Status

- Remediate Situation

- Node Monitor Period = 5s

- Node Monitor Grace Period = 40s

- POD Eviction Timeout = 5m


#### View kube-controller-manager – kubeadm

`kubectl get pods -n kube-system`


 To view the manifest of kube-controller-manager (kubeadm setup): cat /etc/kubernetes/manifests/kube-controller-manager.yaml

To view options of kube-controller-manager (non-kubeadm setup):  

```powershell
cat /etc/system/system/kube-controller-manager.service
ps -aux | grep kube-controller-manager
```

## Kube Scheduler

Control plane component that watches for newly created Pods with no assigned node, and selects a node for them to run on.

1. Filter Nodes

2. Rank Nodes


Factors considered for scheduling decisions include:

1.    Individual and collective resource requirements

2.    Hardware/software/policy constraints

3.    Affinity and anti-affinity specifications

4.    Data Locality

5.    Inter-workload interference

6.    Deadlines

#### View kube-scheduler – kubeadm

`kubectl get pods -n kube-system`


 To view the manifest of kube-scheduler (kubeadm setup): cat /etc/kubernetes/manifests/kube- scheduler.yaml

To view options of kube-scheduler (non-kubeadm setup): 

```powershell
cat /etc/system/system/kube- scheduler.service
ps -aux | grep kube- scheduler
```

## Kubelet

An agent that runs on each node in the cluster. It makes sure that containers are running in a Pod.

The kubelet takes a set of PodSpecs that are provided through various mechanisms and ensures that the containers described in those PodSpecs are running and healthy. The kubelet doesn't manage containers which were not created by Kubernetes.

Kubeadm does not deploy Kubelets.

`ps -aux | grep kubelet`

## Kube Proxy

kube-proxy is a network proxy that runs on each node in your cluster, implementing part of the Kubernetes Service concept.

kube-proxy maintains network rules on nodes. These network rules allow network communication to your Pods from network sessions inside or outside of your cluster.

kube-proxy uses the operating system packet filtering layer if there is one and it's available. Otherwise, kube-proxy forwards the traffic itself.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image5.svg)

`ps -aux | grep kube-proxy `

### PODs

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

A Pod (as in a pod of whales or pea pod) is a group of one or more containers, with shared storage/network resources, and a specification for how to run the containers.

A Pod's contents are always co-located and co-scheduled and run in a shared context. A Pod models an application-specific "logical host": it contains one or more application containers which are relatively tightly coupled. 

#### Using PODs

Usually, Pods are not required to be created directly, even singleton Pods. 

> *Instead, create them using workload resources such as Deployment or Job.* 
> *If your Pods need to track state, consider the StatefulSet resource.*

Pods in a Kubernetes cluster are used in two main ways:

1.    **Pods that run a single container**

2.    **Pods that run multiple containers that need to work together:** A Pod can encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. (**Multi-container PODs**)

#### Pod Definition YAML

```yaml
apiVersion: v1 # version of Kubernetes API
kind: Pod # kind of Kubernetes object to be created
metadata: # data about the Kubernetes object `kind`
  name: myapp-pod
  labels:
    app: myapp
    type: front-end
    env: test
spec: # specification about Kubernetes object `kind`
  containers:
    - name: nginx-container
      image: nginx
  nodeName: # by default is empty
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule" 
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: size
            operator: In
            values:
            - Large
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
  limits:
    memory: "2Gi"
    cpu: 2
    
  schedulerName: my-custom-scheduler
```

kubectl pod-creation command (using yaml): `kubectl create -f pod-definition.yml`
kubectl pod-creation command:  `kubectl run nginx --image=nginx`
kubectl pod-get command: `kubectl get pods`
kubectl pod-deletion command: `kubectl delete pod nginx`

To get information about all the pods: `kubectl get pods`
To describe all the creation information of a pod: `kubectl describe pod myapp-pod`  

A *traditional* container provides several forms of isolation:

- Resource isolation,
- Process isolation,
- Filesystem and mount isolation,
- Network isolation.

The tools that are used under the hood are `Linux namespaces` and `control-groups (cgroups)`.

**`Control groups` are a convenient wat to limit resources such as CPU or memory that a particular process can use.**

**`Namespaces` are in charge of isolating the process and limiting what it can see.**

On Kubernetes, a container provides all of those forms of isolation except network isolations; instead network isolation happens at the pod level.

### ReplicaSets

A ReplicaSet's purpose is to maintain a stable set of replica Pods running at any given time. As such, it is often used to guarantee the availability of a specified number of identical Pods.

A ReplicaSet is defined with fields, including a selector that specifies how to identify Pods it can acquire, several replicas indicating how many Pods it should be maintaining, and a pod template specifying the data of new Pods it should create to meet the number of replicas criteria.

A ReplicaSet then fulfills its purpose by creating and deleting Pods as needed to reach the desired number. When a ReplicaSet needs to create new Pods, it uses its Pod template.

A ReplicaSet ensures that a specified number of pod replicas are running at any given time. However, a Deployment is a higher-level concept that manages ReplicaSets and provides declarative updates to Pods along with a lot of other useful features.

**ReplicaSets**

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
  annotations:
    buildversion: 1.34
spec:
  template:
    metadata:       
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:          
      containers:
        - name: nginx-container
          image: nginx
  replicas: 4
  selector: 
    matchLabels:
      type: front-end
```

**ReplicationController**

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:       
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:          
      containers:
        - name: nginx-container
          image: nginx
  replicas: 4
```


Commands:

1.    To create a replicaset: `kubectl create -f replicaset-definition.yml`

2. To get all replicasets: `kubectl get replicaset`

#### Labels and Selectors

Using labels and selectors, the ReplicaSet knows which pods to monitor in a cluster of pods.

#### Scaling Replicas   

1.    Update the yaml and run the command: `kubectl replace -f replicaset-definition.yml`
2.    We can also use the command: `kubectl scale --replicas=6 -f replicaset-definition.yml`, 
 or, `kubectl scale --replicas=6 -f replicaset myapp-replicaset` but this does not change **replicaset-definition.yml.**

The only difference between replica set and replication controller is the selector types.

The replication controller supports equality based selectors whereas the replica set supports equality based as well as set based selectors.

### Deployments

A Deployment provides declarative updates for Pods and ReplicaSets.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image6.svg)

The contents of **deployment-definition.yml** is as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  template:
    metadata:       
      name: myapp-pod
      labels:
        app: myapp
        type: front-end
    spec:          
      containers:
        - name: nginx-container
          image: nginx
  replicas: 4
  selector: 
    matchLabels:
      type: front-end
```

Commands:

1.    To create a deployment: `kubectl create -f deployment-definition.yml`

2.    To get deployments: `kubectl get deployments`

> BONUS TIP:
>
> 1.    Create a NGINX Pod: `kubectl run nginx --image=nginx`
>
> 2.    Generate POD Manifest YAML file (-o yaml). Don't create it (--dry-run): `kubectl run nginx --image=nginx --dry-run=client -o yaml`
>
> 3.    Create a deployment: `kubectl create deployment --image=nginx nginx`
>
> 4.    Generate Deployment YAML file (-o yaml). Don't create it (--dry-run): `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml`
>
> 5.    Generate Deployment YAML file (-o yaml). Don't create it (--dry-run) with 4 Replicas (--replicas=4): `kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml`
>
> 6.    Save it to a file, make necessary changes to the file (for example, adding more replicas) and then create the deployment.

### Namespaces

Kubernetes supports multiple virtual clusters backed by the same physical cluster. These virtual clusters are called namespaces.

- Namespaces are intended for use in environments with many users spread across multiple teams, or projects. For clusters with a few to tens of users, you should not need to create or think about namespaces at all. Start using namespaces when you need the features they provide.

- Namespaces provide a scope for names. Names of resources need to be unique within a namespace, but not across namespaces. Namespaces cannot be nested inside one another and each Kubernetes resource can only be in one namespace.

- Namespaces are a way to divide cluster resources between multiple users (via resource quota).

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core Concepts/assets/image7.svg)

The `namespace-dev.yml` is follows:

```yaml
apiVersion: v1
kind: Namespace
metadata: 
  name: dev
```

Commands:

1.    To create a pod in a specific namespace: `kubectl create -f pod-definition.yml --namespace=dev`

2.    To create a namespace (with manifest): `kubectl create -f namespace-dev.yml`

3.    To create a namespace (without manifest): `kubectl create namespace dev`

4.    To set the default namespace: `kubectl config set-context $(kubectl config current-context) --namespace=dev`

5.    To get all namespaces: `kubectl get pods --all-namespaces`

#### Resource Quota

The **compute-quota.yml** is as follows:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

Commands:

1.    To create a resource-quota (with manifest): `kubectl create -f compute-quota.yml`

### Services

An abstract way to expose an application running on a set of Pods as a network service, providing :

- Stable IP address;
- Load balancing;
- Loose coupling;
- Within & outside cluster

With Kubernetes you don't need to modify your application to use an unfamiliar service discovery mechanism. Kubernetes gives Pods their own IP addresses and a single DNS name for a set of Pods, and can load-balance across them.

Also, one point to note that, whenever we create a `service`, `kubenetes` creates `endpoint` in the cluster, 

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image8.svg)

#### Service Types:

1.    ClusterIP    
2.    Headless
3.    NodePort
4.    LoadBalancer

#### Services ClusterIP (default type of a service)

A ClusterIP service is the Kubernetes service which gives you a service inside your cluster that other apps inside you cluster can access, for which there is no external access.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core Concepts/assets/image22.svg)

The **clusterip-service-definition.yaml** is as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-internal-service
spec:
  type: ClusterIP
  ports:
    - name: http
      port: 80			# port where the service is exposed
      targetPort: 80		# port where the backend is exposed
      protocol: TCP
  selector:
    app: myapp
    type: back-end
```

You can't access a ClusterIP service from the internet. To do it, you need a Kubernetes proxy.

```powershell
kubectl proxy --port=8080
```

Now, you can navigate through the Kubernetes API to access this service using this scheme:

`http://localhost:8080/api/v1/proxy/namespaces/<NAMESPACE>/services/<SERVICE-NAME>:<PORT-NAME>/`

So, to access the service we defined above, one can use

```powershell
http://localhost:8080/api/v1/proxy/namespaces/<NAMESPACE>/services/<SERVICE-NAME>:<PORT-NAME>/
```

**Use-case:**

1. Debugging your services, or connecting to them directly from your laptop for some reason.
2. Allowing internal traffic, displaying internal dashboards, etc.

**A scenario for using `ClusterIP` service**

![](D:\Work\teaching-myself\learning-k8s\Core Concepts\assets\image23.svg)

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: ms-one-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
  labels:
      name: myingress
spec:
  rules:
  - host: localhost
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            serviceName: ms-one-service
            servicePort: 3200
```

```yaml
# svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: ms-one-service
spec:
  selector:
    app: ms-one
  ports:
  - port: 3200						# arbitrary as 
    targetPort: 3000				# should be same as pod exposed port
```

```powershell
kubectl get endpoints
NAME         ENDPOINTS           AGE
grpc-svc     <none>              89m
```

**What if there is pod is a multi-container pod??**

Let's say there is a pod which is exposed to `27017` and `9216`. 
How to make a ClusterIP service for this type of pod?? 

Basically we have to name those ports.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - name: mongodb
      protocol: TCP
      port: 27017
      targetPort: 27017
    - name: mongodb-exporter
      protocol: TCP
      port: 9216
      targetPort: 9216
```

#### Services Headless

Use-cases of headless services:

- Client wants to communicate with *1 specific Pod* directly.
- Pods want to talk *directly with specific Pod*.
- Stateful applications like `mysql`.

The main problem here is that we need to figure out IP addresses of each Pod.

Option 1 - API call to K8s API server ❓

- makes app too tied to K8s API 🙄
- inefficient 🤭

Option 2 - DNS Lookup

- DNS Lookup for service - returns single IP address (ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service-headless
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - name: mongodb
      protocol: TCP
      port: 27017
      targetPort: 27017
```

No `clusterIP` is assigned.

**Using we run a `headless` service along-with a `ClusterIP` service, so that normal requests go to the `ClusterIP` service and `headless` service will handle specific-pod requests like data-synchronization.**

#### Services NodePort

It creates a service which is accessible on a static port on each worker node in the cluster. The difference between `NodePort` and `ClusterIP` is that `ClusterIP` service is only accessible within the cluster, so no external traffic can directly address the `ClusterIP` service, whereas the `NodePort` service makes the traffic accessible on static or fixed port on each worker node.

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core Concepts/assets/image11.svg)

The **nodeport-service-definition.yml** is as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 8080    # port on target container
      port: 8080          # port on service
      nodePort: 30080   # port on node
  selector:
    name: simple-webapp
```

❗ `nodePort` can be within the range 30000-32767 only.

Another example,

![](D:\Work\teaching-myself\learning-k8s\Core Concepts\assets\image24.svg)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ms-service-nodeport
spec:
  type: NodePort
  selector:
    app: ms-one
  ports:
    - port: 3200
      targetPort: 3000
      nodePort: 30008
```

Commands:

1.    To create a service: `kubectl create -f service-definition.yml`

2.    To get services: `kubectl get services`

#### Services LoadBalancer

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image12.svg)

Typical solution:

Instead we can use native load balancer.

The **loadbalancer-service-definition.yml** is as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  selector:
  	app: ms-one
  ports:
    - targetPort: 3000    # port where the backend is exposed
      port: 3000          # port where the service is exposed
      nodePort: 30010
```

### Kubernetes Imperative and Declarative

#### Imperative Commands

##### Create objects

```powershell
kubectl run --image=nginx nginx  kubectl create deployment --image=nginx nginx  kubectl expose deployment nginx --port 80

kubectl run httpd --image=httpd:alpine --port=80 --expose
```

##### Update Objects

```powershell
kubectl edit deployment nginx
kubectl label pods labelex owner=michael
kubectl scale deployment nginx --replicas=5
kubectl set image deployment nginx nginx=nginx:1.18
kubectl create -f nginx.yml
kubectl replace -f nginx.yml
kubectl delete -f nginx.yml
```


#### Declarative Commands

`````powershell
kubectl apply -f nginx.yaml
```

**Create a deployment :** 

```powershell
kubectl create deployment --image=nginx nginx
```

Generate Deployment YAML file (-o yaml). Don't create it(--dry-run).

```powershell
kubectl create deployment --image=nginx nginx --dry-run=client -o yaml
```

> IMPORTANT:
>
> `kubectl create deployment` does not have a --replicas option. You could first create it and then scale it using the kubectl scale command.
>
> Save it to a file - (If you need to modify or add some other details)
>
> ```powershell
> kubectl create deployment --image=nginx nginx --dry-run=client -o yaml > nginx-deployment.yaml
> ```
>
> You can then update the YAML file with the replicas or any other field before creating the deployment.

**Service**

Create a Service named redis-service of type ClusterIP to expose pod redis on port 6379

```powershell
kubectl expose pod redis --port=6379 --name redis-service --dry-run=client -o yaml
```

(This will automatically use the pod's labels as selectors)

Or

```powershell
kubectl create service clusterip redis --tcp=6379:6379 --dry-run=client -o yaml  
```

(This will not use the pods labels as selectors, instead it will assume selectors as app=redis. You cannot pass in selectors as an option. So, it does not work very well if your pod has a different label set. So, generate the file and modify the selectors before creating the service)

Create a Service named nginx of type NodePort to expose pod nginx's port 80 on port 30080 on the nodes:

```powershell
kubectl expose pod nginx --port=80 --name nginx-service --type=NodePort --dry-run=client -o yaml
```

(This will automatically use the pod's labels as selectors, but you cannot specify the node port. You have to generate a definition file and then add the node port in manually before creating the service with the pod.)

Or

```powershell
kubectl create service nodeport nginx --tcp=80:80 --node-port=30080 --dry-run=client -o yaml
```

(This will not use the pods labels as selectors)


Both the above commands have their own challenges. While one of it cannot accept a selector the other cannot accept a node port. I would recommend going with the `kubectl expose` command. If you need to specify a node port, generate a definition file using the same command and manually input the nodeport before creating the service.

 

