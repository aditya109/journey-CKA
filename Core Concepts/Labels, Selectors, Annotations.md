# Labels, Selectors, Annotations

## Labels

prefix/name=value

Example, `dev.vivek.something/tier=frontend`

The general format in the label is kept so that the  username is prefixed to the label key name to as to differentiate between any existing labels.

To label any resource in Kubernetes, we can use :
`kubectl label` command.

Let's create a deployment `nginx`.

```sh
kubectl create deployment --image nginx nginx
```

To label this deployment:

```sh
kubectl label deployment.apps nginx tier=frontend
```

This is `-oyaml` of the above looks like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-10-30T12:37:25Z"
  generation: 1
  labels:
    app: nginx
    daitya96/tier: frontend					👈	
  name: nginx
  namespace: default
  resourceVersion: "33308"
  uid: 7eafb764-cd46-4d37-bc3a-df33a905056a
spec:
  progressDeadlineSeconds: 600
  replicas: 1
 ......
```

To select resources with the given label, we do:

- Equality-based comparison

  ```sh
  kubectl get pods -l tier=frontend
  kubectl get pods -l tier!=app
  ```

- Set-based operation 

  ```sh
  kubectl get pods -l 'app in (frontend, nginx)'
  kubectl get pods -l 'app notin (frontend, nginx)'
  kubectl get pods -l app # all pods with `app` label key 
  kubectl get pods -l !app # all pods without `app` label key 
  ```

  > AND operator:
  >
  > ```sh
  > kubectl get pods -l app=nginx,tier=frontend # comma means AND operator
  > ```

## Usecases of Labels in manifests

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2021-10-30T12:37:25Z"
  generation: 2
  labels:
    app: nginx										# 👈 deployment label; can be different from below two
  name: nginx
  namespace: default
  resourceVersion: "34170"
  uid: 7eafb764-cd46-4d37-bc3a-df33a905056a
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx									# 👈 label of pods which this deployment has to manage
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx									# 👈 pod label
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2021-10-30T12:37:25Z"
    lastUpdateTime: "2021-10-30T12:37:38Z"
    message: ReplicaSet "nginx-6799fc88d8" has successfully progressed.
    reason: NewReplicaSetAvailable
    status: "True"
    type: Progressing
  - lastTransitionTime: "2021-10-31T04:10:48Z"
    lastUpdateTime: "2021-10-31T04:10:48Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  observedGeneration: 2
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

Similarly, we are exposing the above deployment,

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
    daitya96/tier: frontend
  name: nginx
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: nginx								# 👈 label of pods which this service has to expose
status:
  loadBalancer: {}
```

The endpoints created by the above:

```sh
>: kubectl get endpoints
NAME         ENDPOINTS                                         AGE
kubernetes   192.168.49.2:8443                                 42d
nginx        172.17.0.3:8080,172.17.0.4:8080,172.17.0.5:8080   2m51s

```


Another example, we can try to create a pod,

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  nodeSelector:
  	this: label					# 👈 label of nodes upon which this particular pod can be created
status: {}
```

But let's say we do not have this node, but still we try to create this pod- it would be in `Pending` state. Let's try to `describe` the pod.

```yaml
Name:         nginx
Namespace:    default
Priority:     0
Node:         <none>
Labels:       run=nginx
Annotations:  <none>
Status:       Pending
IP:           
IPs:          <none>
Containers:
  nginx:
    Image:        nginx
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jwz7v (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-jwz7v:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              this=label
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  4s    default-scheduler  0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.

```























