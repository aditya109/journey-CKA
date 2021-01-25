# Table of Contents

- [Scheduling](#scheduling)
  * [Manually Scheduling](#manually-scheduling)
  * [No Scheduler](#no-scheduler)
  * [Labels and Selectors](#labels-and-selectors)
    + [Annotations](#annotations)
  * [Taints and Tolerations](#taints-and-tolerations)
    * [Taint Effects:](#taint-effects-)
  * [Node Selectors](#node-selectors)
  * [Node Affinity](#node-affinity)
  * [Resources Requirements and Limits](#resources-requirements-and-limits)
    + [Resource Limits](#resource-limits)
    + [Limit Range](#limit-range)
      - [A quick note on editing PODs and Deployments](#a-quick-note-on-editing-pods-and-deployments)
        * [Edit a POD](#edit-a-pod)
        * [Edit Deployments](#edit-deployments)
  * [Daemon Sets](#daemon-sets)
  * [Static Pods](#static-pods)
  * [Multiple Schedulers](#multiple-schedulers)
    + [Custom Scheduler](#custom-scheduler)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

# Scheduling

##        Manually Scheduling

The scheduler goes through all the nodes and finds those nodes who’s this property is not set. These are the nodes which will be used for scheduling. Once identified, it sets this property of the node in the pod along side with it and schedules the pod on that node, creating a binding object.

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
  nodeName: # by default is empty 👈
```

## No Scheduler

Create a **pod-bind-definition.yml** and manually schedule the pods on the node.

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: nginx
target: 
  apiVersion: v1
  kind: Node
  name: node02
```

Then send a POST request to binding API:

```bash
curl --header "Content-Type:application/json" --request POST --data '{"apiVersion": "v1", "kind":"Binding", ...}' http://$SERVER/api/v1/namespaces/default/pods/$PODNAME/binding/
```

This will schedule the pods which label on the specified node.

## Labels and Selectors

Using selectors to select the pod with given label:

```powershell
kubectl get pods --selectors app=App1
```

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
        app: myapp				👈👈👈👈👈👆
        type: front-end					   👆 
    spec:          						   👆
      containers:						   👆
        - name: nginx-container			   👆
          image: nginx					   👆
  replicas: 4							   👆
  selector: 							   👆
    matchLabels:						   👆
      type: front-end  			👉👉👉👉👉👆
```



## Annotations

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
    type: front-end
  annotations:
    buildversion: 1.34					👈
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



## Taints and Tolerations

They are used to restrict the pod to be scheduled to particular nodes.

*Taints* allow a node to repel a set of pods. *Tolerations* are applied to pods, and allow the pods to schedule onto nodes with matching taints.

 Taints and tolerations work together to ensure the pods are not scheduled onto inappropriate nodes. One or more taints are applied to a node; this marks that the node should not accept any pods that do not tolerate the taints.

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
    - key: "app"					👈
      operator: "Equal"
      value: "blue"					👈
      effect: "NoSchedule" 			👈
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



**Taints – Node:** `kubectl taint nodes node-name key-value: taint-effect`

```powershell
kubectl taint nodes node1 app=blue :NoSchedule
```

##### Taint Effects:

In particular,

- if there is at least one un-ignored taint with effect **NoSchedule** then Kubernetes will not schedule the pod onto that node

- if there is no un-ignored taint with effect NoSchedule but there is at least one un-ignored taint with effect **PreferNoSchedule** then Kubernetes will try to not schedule the pod onto the node

- if there is at least one un-ignored taint with effect **NoExecute** then the pod will be evicted from the node (if it is already running on the node), and will not be scheduled onto the node (if it is not yet running on the node).


**Removing Tolerations – Pods:**

```powershell
kubectl taint nodes master/controlplane node-role.kubernetes.io/master:NoSchedule
```

## Node Selectors

**nodeSelector** is the simplest recommended form of node selection constraint. **nodeSelector** is a field of PodSpec. For the pod to be eligible to run on a node, the node must have each of the indicated key-value pairs as labels (it can have additional labels as well). The most common usage is one key-value pair.

1. Run `kubectl get nodes` to get the names of your cluster’s nodes.

2. Pick out the one that you want to add a label to , and then run `kubectl label nodes <node-name> <label-key>=<label-value>` to add a label to the node you’ve chosen. 
    E.g., `kubectl label nodes kubernetes-foo-node-1.c.a-robinson.internal disktype=ssd`

3. You can verify that it worked by re-running `kubectl get nodes --show-labels` and checking that the node now has a label. You can also use kubectl describe node `nodename` to see the full list of labels of the given node. 

4. Take whatever pod config file you want to run, and add a `nodeSelector` section to it, like this. 

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
     nodeSelector:
       size: Large								👈
     resources:
       requests:
         memory: "1Gi"
         cpu: 1
     limits:
       memory: "2Gi"
       cpu: 2
       
     schedulerName: my-custom-scheduler
   ```

5. When you then run `kubectl apply -f pod-definition.yaml`, the Pod will get scheduled on the node that you attached the label to. You can verify that it worked by running `kubectl get pods -o wide` and looking at the "NODE" that the Pod was assigned to.

## Node Affinity

Node affinity is conceptually similar to nodeSelector -- it allows you to constrain which nodes your pod is eligible to be scheduled on, based on labels on the node.

There are currently two types of node affinity, as follows:

1. `requiredDuringSchedulingIgnoredDuringExecution` only run the pod on nodes with Intel CPUs",

2. `preferredDuringSchedulingIgnoredDuringExecution` try to run this set of pods in failure zone XYZ, but if it's not possible, then allow some to run elsewhere. 

## Resources Requirements and Limits

By default, Kubernetes provides **0.5 CPU** and **256 Mi** of memory. This is known as the resource request for a container.

 Modification in resource requirements are provided in **pod-definition.yml:**

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
  nodeSelector:
    size: Large
  resources:				👈
    requests:
      memory: "1Gi"
      cpu: 1
  limits:
    memory: "2Gi"
    cpu: 2
    
  schedulerName: my-custom-scheduler
```



**CPU count** can be as low as 0.1(100m) or 1m.

1 CPU count = 1 AWS vCPU = 1 GCP Core = 1 Azure Core = 1 Hyperthread


 **Memory Count:**

 ```
1 G/Gigabyte = 1000000000 bytes
1 M/Megabyte = 1000000 bytes
1 K/Kilobyte = 1000 bytes

1 Gi/Gibibyte = 1,073,741,824 bytes
1 Mi/Mebibyte = 1,048,576 bytes
1 Ki/Kibibyte = 1,024 bytes
 ```

### Resource Limits

By default, the Kubernetes has a limit of 1 vCPU and 512 Mi memory.


To change it, just mention in the **pod-definition.yml.**

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
  nodeSelector:
    size: Large
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
  limits:				👈
    memory: "2Gi"
    cpu: 2
    
  schedulerName: my-custom-scheduler
```

If a pod tries to exceed CPU usage limit, the node gets throttled, but does not allow excessive memory usage.

This is not the case with memory. The pod can exceed its memory limit, but it constantly uses excessive memory it will get terminated.

### Limit Range

> When a pod is created the containers are assigned a default CPU request of .5 and memory of 256Mi. 

For the POD to pick up those defaults you must have first set those as default values for request and limit by creating a LimitRange in that namespace. 

 ```yaml
apiVersion: v1
kind: LimitRange
metadata: 
  name: mem-cpu-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 1
    defaultRequest:
      memory: 256Mi
      cpu: 0.5
    type: Container
 ```



#### A quick note on editing PODs and Deployments

##### Edit a POD

We cannot edit specifications of an existing POD other than the below.

1. `spec.containers[*].image`

2. `spec.initContainers[*].image`

3. `spec.activeDeadlineSeconds`

4. `spec.tolerations`

##### Edit Deployments

With Deployments you can easily edit any field/property of the POD template. Since the pod template is a child of the deployment specification, with every change the deployment will automatically delete and create a new pod with the new changes. So, if you are asked to edit a property of a POD part of a deployment you may do that simply by running the command

```powershell
kubectl edit deployment my-deployment
```

## Daemon Sets

A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. As nodes are added to the cluster, Pods are added to them. As nodes are removed from the cluster, those Pods are garbage collected. Deleting a DaemonSet will clean up the Pods it created.

Some typical uses of a DaemonSet are:

- running a cluster storage daemon on every node

- running a logs collection daemon on every node

- running a node monitoring daemon on every node

 It uses `NodeAffinity` to schedule pods on nodes.

 ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Core%20Concepts/assets/image21.svg)

 ```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: monitoring-daemon
spec:
  containers:
  - name: monitoring-agent
    image: monitoring-agent
  selector: 
    matchLabels:
      app: monitoring-agent
  template:
    metadata:
      labels:
        app: monitoring-agent
 ```

Commands:

To get daemon sets: `kubectl get daemonsets`
To describe daemonsets: `kubectl describe daemonsets monitoring-daemon`

## Static Pods

Static Pods are managed directly by the kubelet daemon on a specific node, without the API server observing them. Unlike Pods that are managed by the control plane (for example, a Deployment); instead, the kubelet watches each static Pod (and restarts it if it fails).

Static Pods are always bound to one Kubelet on a specific node.

The kubelet automatically tries to create a mirror Pod on the Kubernetes API server for each static Pod. This means that the Pods running on a node are visible on the API server, but cannot be controlled from there.

In a lone node, there is location called pod-manifest from where `/etc/kubernetes/manifests` the kubelet periodically checks, and creates the pods as static pods (only pods can be created by kubelet), which can be changed in `kubelet.service` by providing `kubeconfig.yaml`.

A change in these yamls leads to recreations of pods, and removal of any of these manifests leads to deletion of pre-existing static pod.

To check for the static pods (with kube-apiserver): `kubectl get pods`
To check for the static pods (without kube-apiserver, with just kubelet: `docker ps` 
To get pod-manifests-directory location: `ps -ef | grep kubelet | grep "\--config"`

| Static PODs                                     | DaemonSets                                            |
| ----------------------------------------------- | ----------------------------------------------------- |
| Created by Kubelet                              | Created by kube-apiserver (via  DaemonSet controller) |
| Deploy Control Plane components as  Static Pods | Deploy Monitoring Agents, Logging  Agents on Nodes    |
| Ignored  by the Kube-Scheduler                  |                                                       |


 Steps to start as a static Pod:

1. Choose a node where you want to run the static Pod.
   `ssh my-node1`

2. Choose a directory, say **/etc/kubelet.d** and place a web server Pod definition there, for example **/etc/kubectl.d/static-web.yaml**.

 Run this command on the node where kubelet is running

```bash
mkdir /etc/kubelet.d/
cat /etc/kubelet.d/static-web.yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: myrole
spec:
  containers:
    - name: web
      image: nginx
      ports:
        - name: web
          containerPort: 80
          protocol: TCP
```

3. Configure your kubelet on the node to use this directory by running it **with --pod-manifest-path=/etc/kubelet.d/ argument**. Add the **<staticPodPath: <the-directory>** field in the kubelet configuration file.

4. Restart the kubelet: `systemctl restart kubelet`

## Multiple Schedulers

To create a **custom-scheduler.service**, use the custom scheduler binary, change the **–scheduler-name** to your own custom scheduler name.

### Custom Scheduler

```yaml
# my-custom-scheduler.yaml

apiVersion: v1
kind: Pod
metadata: 
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    - --scheduler-name=my-custom-scheduler
    - --lock-object-name=my-custom-scheduler

    image: k8s.gcr.io/kube-scheduler-amd64:v1.11.3
    name: kube-scheduler
```

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
  nodeSelector:
    size: Large
  resources:
    requests:
      memory: "1Gi"
      cpu: 1
  limits:
    memory: "2Gi"
    cpu: 2
    
  schedulerName: my-custom-scheduler				👈
```

```bash
kubectl get events
kubectl logs my-custom-scheduler --name-space=kube-system
```



 

 