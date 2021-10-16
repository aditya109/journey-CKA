# Application Lifecycle Management

## Contents

- [Rolling Updates and Rollbacks in Deployments](#rolling-updates-and-rollbacks-in-deployments)
  * [Rollout and Versioning](#rollout-and-versioning)
  * [Deployment Strategy](#deployment-strategy)
  * [How to apply an update ??](#how-to-apply-an-update---)
  * [Rollback](#rollback)
- [Configure Applications](#configure-applications)
  * [Application Commands](#application-commands)
  * [Application - Commands and Arguments](#application---commands-and-arguments)
  * [Configuring Environment Variables in Applications](#configuring-environment-variables-in-applications)
    + [Using `ConfigMaps`](#using--configmaps-)
    + [Use `Secrets`](#use--secrets-)
      - [Imperative way of creating a `secret`](#imperative-way-of-creating-a--secret-)
      - [Declarative way of creating a `secret`](#declarative-way-of-creating-a--secret-)
- [Multi Container PODs](#multi-container-pods)
  * [Securing an HTTP service](#securing-an-http-service)
    + [Types of multi-container patterns](#types-of-multi-container-patterns)
      - [***Ambassador Pattern***: *(as Proxy containers)*](#---ambassador-pattern-------as-proxy-containers--)
        * [How Ambassador pattern works ?](#how-ambassador-pattern-works--)
      - [*Adapter Pattern:* *(exposing metrics with a standard interface)*](#-adapter-pattern-----exposing-metrics-with-a-standard-interface--)
      - [*Sidecar Pattern: (Tailing logs)*](#-sidecar-pattern---tailing-logs--)
- [initContainers](#initcontainers)
  * [Differences from regular containers](#differences-from-regular-containers)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

## Rolling Updates and Rollbacks in Deployments

### Rollout and Versioning

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/rollouts.svg)

Suppose we have multiple sets of applications running in our cluster, and let's say we are on *version* `Revision 1` of the application. We now have to upgrade all of those running applications to `Revision 2`. The process of upgrading all our running applications to required version is called **rollout**.

```powershell
kubectl rollout status deployment/myapp-deployment
```

The above command shows the status of the pushed rollout.

```powershell
kubectl rollout history deployment/myapp-deployment
```

The above command shows the revisions and history of our deployment.

### Deployment Strategy

> Setup: We have 4 running pods, on each runs our application.

There are popularly 2 strategies used in deployments:

- Bring all 4 instances of our application down, and then bring 4 new instances up to replace them. This is called **Recreate** strategy.
  ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/st1.svg)
  The problem with this strategy is if something goes wrong in the update process, we neither have any running instances to cater to existing users, nor do we have the new instances, (as the something could have gone wrong, or amidst update process), there is no running instance for catering users, which is really **BFB** (Bad for Business).

- Bring alternative instances of our application down, while keeping the left-out instances running on older version, and then bring new instances to replace downed instances. This is called **Rolling Update** strategy.
  ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/assets/st2.svg)

### How to apply an update ??

Let's take an example that this is our application deployment yaml manifest. This is our Revision 1 manifest.

```yaml
# deployment-definition.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  selector:
    matchLabels:
      app: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp-pod
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx						👈
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
  replicas: 3
```

In order to update, we just have to update image name from `nginx` to `nginx:1.7.1` and that's it. This is our Revision 2 manifest.

```yaml
# deployment-definition.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
    type: front-end
spec:
  selector:
    matchLabels:
      app: front-end
  template:
    metadata:
      name: myapp-pod
      labels:
        app: myapp-pod
        type: front-end
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.7.1					👈
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
  replicas: 3
```

In order to apply this upgrade, we type:

```powershell
kubectl apply -f deployment-definition.yml
```

This triggers a new rollout and a new revision is created.

Also, quick tip, to update the image we could also do the following:  

```powershell
kubectl set image deployment/myapp-deployment nginx=nginx:1.7.1
```

> `RollingUpdate`  is K8s-default for upgrades.

### Rollback

To undo a change, just type:

```powershell
kubectl rollout undo deployment/myapp-deployment
```

> `RollingUpdate`  is K8s-default for upgrades.

## Configure Applications

### Application Commands

Whenever when load some Docker image, let's say, for example, `ubuntu`, it immediately follows certain protocols and create a container and exits it.

```powershell
🐳 » docker run ubuntu
🐳 » docker ps 
CONTAINER ID   IMAGE   COMMAND   CREATED   STATUS   PORTS   NAMES
```

Still if we do `docker ps -a`, we would see the container details.

```powershell
Aditya :: system32 » docker ps -a
CONTAINER ID   IMAGE                       COMMAND                  CREATED              STATUS                      PORTS     NAMES
d227eb4f969d   ubuntu                      "/bin/bash"              About a minute ago   Exited (0) 59 seconds ago             practical_pike
```

This is because of `CMD` instruction in `Dockerfile` of the `ubuntu` image.

```dockerfile
FROM scratch
ADD ubuntu-focal-core-cloudimg-amd64-root.tar.gz /

..............

# verify that the APT lists files do not exist
RUN [ -z "$(apt-get indextargets)" ]
# (see https://bugs.launchpad.net/cloud-images/+bug/1699913)

# make systemd-detect-virt return "docker"
# See: https://github.com/systemd/systemd/blob/aa0c34279ee40bce2f9681b496922dedbadfca19/src/basic/virt.c#L434
RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container

CMD ["/bin/bash"]           👈
```

As we can see `bash` is a shell not a process, it listens to input on the `stdout`, but if it doesn't find any it exits out.

To solve this issue, we could potentially add a command in the end of **run** command `docker run ubuntu sleep 5` or use our own custom `Dockerfile`.

```dockerfile
FROM ubuntu

CMD sleep 5
```

We can also specify the `Dockerfile` in the following manner:

```dockerfile
# Dockerfile
FROM ubuntu

CMD ["sleep", "5"]         # CMD ["command", "param1", "param2"]
```

```powershell
🐳 » docker build -t ubuntu-sleeper .
[+] Building 0.2s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                        0.0s
 => => transferring dockerfile: 72B                                                                                                                                                                         0.0s
 => [internal] load .dockerignore                                                                                                                                                                           0.0s
 => => transferring context: 2B                                                                                                                                                                             0.0s
 => [internal] load metadata for docker.io/library/ubuntu:latest                                                                                                                                            0.0s
 => [1/1] FROM docker.io/library/ubuntu                                                                                                                                                                     0.1s
 => => resolve docker.io/library/ubuntu:latest                                                                                                                                                              0.0s
 => exporting to image                                                                                                                                                                                      0.0s
 => => exporting layers                                                                                                                                                                                     0.0s
 => => writing image sha256:bf9d10e69adc13eb242d05de6c9fa1e9c9b4caef6d9c620950442df511e47238                                                                                                                0.0s
 => => naming to docker.io/library/ubuntu-sleeper
```

```powershell
🐳 » docker run ubuntu-sleeper
```

How to send seconds on `sleep` as command-line arguments ?

For that, we use `ENTRYPOINT ` instruction. Whenever we run, `ENTRYPOINT` acts like `CMD`, but uses command-line arguments as its parameters.

```dockerfile
# Dockerfile
FROM ubuntu

ENTRYPOINT ["sleep"]        
```

```powershell
🐳 »  docker build -t ubuntu-sleeper .
[+] Building 0.1s (5/5) FINISHED
 => [internal] load build definition from Dockerfile                                              0.0s
 => => transferring dockerfile: 73B                                                               0.0s
 => [internal] load .dockerignore                                                                 0.0s
 => => transferring context: 2B                                                                   0.0s
 => [internal] load metadata for docker.io/library/ubuntu:latest                                  0.0s
 => CACHED [1/1] FROM docker.io/library/ubuntu                                                    0.0s
 => exporting to image                                                                            0.0s
 => => exporting layers                                                                           0.0s
 => => writing image sha256:9e12bfb37adca3310efa3de95d484541d8818860af830542a97849551ffd9585      0.0s
```

```powershell
🐳 » docker run ubuntu-sleeper 10
```

This command here `docker run ubuntu-sleeper 10`  acts like `docker run ubuntu-sleeper sleep 10`. But there an issue here. What if the user gave `docker run ubuntu-sleeper` ? That would give an error as `sleep` requires a parameter, which is clearly not provided. How to fix this ??

We can provided a default argument in `CMD`.

```dockerfile
# Dockerfile
FROM ubuntu

ENTRYPOINT ["sleep"]   

CMD ["5"]
```

Here, `5` gets appended to `sleep` automatically if nothing is provided as an argument. So, a command like `docker run ubuntu-sleeper` would become equivalent to `docker run ubuntu-sleeper sleep 5` and `docker run ubuntu-sleeper 10`  would become equivalent to `docker run ubuntu-sleeper sleep 10`.

One last thing, how to override `ENTRYPOINT` instructions of the Dockerfile? 

```powershell
	🐳 » docker run --name ubuntu-sleeper \
			--entrypoint sleep2.0
			ubuntu-sleeper 10
```

### Application - Commands and Arguments

But how would we create a `yml` file for this image ?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntu-sleeper-pod
  labels:
    name: ubuntu-sleeper-pod
spec:
  containers:
  - name: ubuntu-sleeper-pod
    image: ubuntu-sleeper
    command: ["sleep2.0"]   # this field overrides the ENTRYPOINT instruction of the Dockerfile
    args: ["10"]            # CMD instruction is overriden here
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
```

### Configuring Environment Variables in Applications

```powershell
🐳 » docker run -e APP_COLOR=pink simple-webapp-color
```

Attempting to do the same in `pod-definition.yml`.

```yaml
# pod-definition-2.yml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
    env:
      - name: APP_COLOR
        value: pink
    # env: # using ConfigMaps 
    #   - name: APP_COLOR
    #     valueFrom: 
    #       confifMapKeyRef: #config map path
    # env: # using Secrets 
    #   - name: APP_COLOR
    #     valueFrom: #secret map path
    #       secretKeyRef:
    #         key: 
```

#### Using `ConfigMaps`

**Imperative method:**

```powershell
🐳 » kubectl create configmap \
			app-config --from-literal=APP_COLOR=blue \
					   --from-literal=APP_MODE=prod
```

```bash
# app_config.properties
APP_COLOR=blue
APP_MODE=prod
```

```powershell
🐳 » kubectl create configmap \
			app-config --from-file=app_config.properties
```

**Declarative method:**

```yaml
# config-map.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: blue
  APP_MODE: prod
```

```powershell
🐳 » kubectl create -f config-map.yaml
configmap/app-config created
🐳 » kubectl get configmaps
NAME         DATA   AGE
app-config   2      13s
🐳 » kubectl describe configmap/app-config
Name:         app-config
Namespace:    default   
Labels:       <none>    
Annotations:  <none>    

Data
====
APP_COLOR:
----
blue
APP_MODE:
----
prod
```

```yaml
# pod-definition-2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp-color
  labels:
    name: simple-webapp-color
spec:
  containers:
  - name: simple-webapp-color
    image: simple-webapp-color
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 8080
    envFrom:
      - configMapRef:
          name: app-config
```

Other ways to inject `configMaps`:

1. ```yaml
   	envFrom:
         - configMapRef:
             name: app-config
   ```

2. ```yaml
       env:
         - name: APP_COLOR
           valueFrom:
             configMapKeyRef:
               name: app-config
               key: APP_COLOR
   ```

3. ```yaml
       volumes:
       - name: app-config-volume
         configMap:
           name: app-config
   ```

> `kubectl get pod webapp-color -o yaml > pod.yaml`

#### Use `Secrets`

##### Imperative way of creating a `secret` 

```bash
kubectl create secret generic \
	<secret-name> --from-literal=<key>=<value>
```

Example,

```bash
kubectl create secret generic \
	app-secret --from-literal=DB_Host=mysql \
				--from-literal=DB_User=root \
				--from-literal=DB_Password=paswrd
```

Files can be used as well,

```bash
kubectl create secret generic \
	<secret-name> --from-file=<path-to-file>
```

Example,

```bash
kubectl create secret generic \
	app-secret --from-file=app_secret.properties
```



##### Declarative way of creating a `secret`

```yaml
# secret-data.yml

apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_Host: mysql
  DB_User: root
  DB_Password: paswrd
```

But this is not safe, we have to convert this to some encoded format to work with it.

```yaml
# secret-data.yml

apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_Host: bQB5AHMAcQBsAA==
  DB_User: cgBvAG8AdAA=
  DB_Password: bQB5AHMAcQBsAA==
```

To convert normal text to `base64`, 

in Windows

```powershell
$MYTEXT = 'root' ;  $ENCODED = [Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($MYTEXT)) ; Write-Output $Encoded
```

in Linux

```bash
echo -n 'root' | base64
```

To create the `secret`, run

```bash
kubectl create -f secret-data.yml
kubetl get secrets

NAME                    TYPE                                  DATA   AGE
app-secret              Opaque                                3      11s
dashboard-token-6vdhl   kubernetes.io/service-account-token   3      75d
default-token-jl8bg     kubernetes.io/service-account-token   3      81d
```

```powershell
kubectl describe secret app-secret  

Name:         app-secret
Namespace:    default   
Labels:       <none>    
Annotations:  <none>    

Type:  Opaque

Data
====
DB_Host:      10 bytes  
DB_Password:  10 bytes  
DB_User:      8 bytes   
```

To view all the values as well,

```powershell
kubectl get secret app-secret -o yaml

apiVersion: v1
data:
  DB_Host: bQB5AHMAcQBsAA==
  DB_Password: bQB5AHMAcQBsAA==
  DB_User: cgBvAG8AdAA=
kind: Secret
metadata:
  creationTimestamp: "2021-04-13T05:48:18Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:DB_Host: {}
        f:DB_Password: {}
        f:DB_User: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-04-13T05:48:18Z"
  name: app-secret
  namespace: default
  resourceVersion: "386643"
  selfLink: /api/v1/namespaces/default/secrets/app-secret
  uid: 3797a0c5-ffb1-4a15-9ec5-f6aefd642ecb
type: Opaque
```

Configuring a pod to use secret

1. Use the `secret` entirely.	

   ```yaml
   # pod-definition-3.yml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-with-secret
     labels:
       name: pod-with-secret
   spec:
     containers:
     - name: pod-with-secret
       image: pod-with-secret
       ports:
         - containerPort: 8080
       envFrom:
         - secretRef:
             name: app-secret
   ```

2. Use selective keys from `secret`.

   ```yaml
   # pod-definition-3.yml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-with-secret
     labels:
       name: pod-with-secret
   spec:
     containers:
     - name: pod-with-secret
       image: pod-with-secret
       ports:
         - containerPort: 8080
       env:
         - name: DB_Password
           valueFrom:
             secretKeyRef:
               name: app-secret
               key: DB_Password
   ```

3. Mount the secret within a volume

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: pod-with-secret
     labels:
       name: pod-with-secret
   spec:
     containers:
     - name: pod-with-secret
       image: pod-with-secret
       ports:
         - containerPort: 8080
       volume:
         - name: app-secret-volume
           secret: 
             secretName: app-secret
   ```

## Multi Container PODs

[Extending applications on Kubernetes with multi-container pods (learnk8s.io)](https://learnk8s.io/sidecar-containers-patterns)

What about running applications that weren't explicitly designed to be run in a containerized environment?
*We can use multi-container pods.*

Why would we want to run multiple containers in a pod?
*Multi-container pods allow we to change the behavior of an application without changing its code.*

### Securing an HTTP service

*Let's use Elasticsearch as an example application that we'd like to enhance using multi-container pods.*

```yaml
# es-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type # The discovery.type environment variable is necessary to get it running with a single replica.
              value: single-node
          ports:
            - name: http
              containerPort: 9200
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
spec:
  selector:
    app.kubernetes.io/name: elasticsearch
  ports:
    - port: 9200
      targetPort: 9200
```

This pod works by running another pod in the cluster and `curl`ing to the `elasticsearch` service.

```bash
kubectl run -it --rm --image=curlimages/curl curl \
  -- curl http://elasticsearch:9200
```

```bash
{
  "name" : "elasticsearch-77d857c8cf-mk2dv",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "z98oL-w-SLKJBhh5KVG4kg",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "we Know, for Search"
}
```

Let's say that we're moving towards a zero-trust security model and we'd like like to encrypt all traffic on the network.

*How would we go about this if the application doesn't have native TLS support?*

Initial idea would be to do TLS termination with an `nginx ingress`, since the ingress is the component routing the external traffic in the cluster, but that won't meet the requirements, as traffic between the `ingress` pod and the `elasticsearch` pod could go over the external traffic in the cluster.

The solution that will meet the requirements is to tack an `nginx` proxy container onto the pod that will listen over TLS.

```yaml
# es-deployment-v1.1.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: elasticsearch
  template:
    metadata:
      labels:
        app.kubernetes.io/name: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: elasticsearch:7.9.3
          env:
            - name: discovery.type
              value: single-node
            - name: network.host
              value: 127.0.0.1
            - name: http.port
              value: '9201'
        - name: nginx-proxy
          image: nginx:1.19.5
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
              readOnly: true
            - name: certs
              mountPath: /certs
              readOnly: true
          ports:
            - name: https
              containerPort: 9200
      volumes:
        - name: nginx-config
          configMap:
            name: elasticsearch-nginx
        - name: certs
          secret:
            secretName: elasticsearch-tls
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: elasticsearch-nginx
data:
  elasticsearch.conf: |
    server {
        listen 9200 ssl;
        server_name elasticsearch;
        ssl_certificate /certs/tls.crt;
        ssl_certificate_key /certs/tls.key;

        location / {
            proxy_pass http://localhost:9201;
        }
    }

The above means the following:

- Elasticsearch is listening on `localhost` on port `9201` instead of the default `0.0.0.0:9200` (that's what the `network.host` and `http.port` environment variables are for).
- The new `nginx-proxy` container listens on port `9200` over HTTPS and proxies requests to Elasticsearch on port `9201`. (The `elasticsearch-tls` secret contains the TLS cert and key, which could be generated with [cert-manager](https://cert-manager.io/), for example.)

We can confirm it's working by making an HTTPS request from within the cluster.

​```bash
kubectl run -it --rm --image=curlimages/curl curl \
  -- curl -k https://elasticsearch:9200
{
  "name" : "elasticsearch-5469857795-nddbn",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "XPW9Z8XGTxa7snoUYzeqgg",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "we Know, for Search"
}
```

> The `-k` version is necessary for self-signed TLS certificates. In a production environment, we'd want to use a trusted certificate.

#### Types of multi-container patterns

- ##### ***Ambassador Pattern***: *(as Proxy containers)* 

  Here are a few other things one can do with the Ambassador Pattern:

  - If we want all the traffic in the cluster to be encrypted with TS certificate, we might decide to install an nginx proxy in every pod in the cluster. We can even go a step farther and use `mutual TLS` to ensure that all requests are authenticated as well as encrypted. (*Primary approach used service meshes such as Istio and Linkerd.*)
  - We can use a proxy to ensure that a centralized OAuth authority authenticates all requests by verifying JWTs. One example of this is `gcp-iap-auth`, which verifies that requests are authenticated by GCP Identity-Aware-Proxy.
  - We can connect over a secure tunnel to an external database. This is especially handy for databases that don't have built-in TLS support. Another example is the Google Cloud SQL Proxy.

  ```yaml
  # es-ambassador.yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: podtest
  spec:
    containers:
      - name: c1
        image: busybox
        command: ['sleep', '5000']
        volumeMounts:
          - name: shared
            mountPath: /shared
      - name: c2
        image: busybox
        command: ['sleep', '5000']
        volumeMounts:
          - name: shared
            mountPath: /shared
    volumes:
      - name: shared
        emptyDir: {}
  ```

  ###### How Ambassador pattern works ?

  - Since all containers share the same network namespace, a single container can listen to all connections – even external ones.
  - The rest of the containers only accept connections from localhost – rejecting any external connection.

  > A point to notes is though, because the network namespace is shared, multiple containers in a pod can't listen on the same port !

- ##### *Adapter Pattern:* *(exposing metrics with a standard interface)*

  Let's say we have standardized on using Prometheus from monitoring all of the services in your Kubernetes cluster, but you're using some applications that don't natively export Prometheus metrics.

  For example, let's add an `exporter` container to the pod that exposes various Elasticsearch metrics in the Prometheus format.

  ```yaml
  # es-prometheus.yml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: elasticsearch
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: elasticsearch
    template:
      metadata:
        labels:
          app.kubernetes.io/name: elasticsearch
      spec:
        containers:
          - name: elasticsearch
            image: elasticsearch:7.9.3
            env:
              - name: discovery.type
                value: single-node
            ports:
              - name: http
                containerPort: 9200
          - name: prometheus-exporter
            image: justwatch/elasticsearch_exporter:1.1.0
            args:
              - '--es.uri=http://localhost:9200'
            ports:
              - name: http-prometheus
                containerPort: 9114
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: elasticsearch
  spec:
    selector:
      app.kubernetes.io/name: elasticsearch
    ports:
      - name: http
        port: 9200
        targetPort: http
      - name: http-prometheus
        port: 9114
        targetPort: http-prometheus
  ```

- ##### *Sidecar Pattern: (Tailing logs)*

  Taking back the Elasticsearch example, which is a bit contrived since the Elasticsearch container logs to standard out by default.

  ```yaml
  # sidecard-example.yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: elasticsearch
    labels:
      app.kubernetes.io/name: elasticsearch
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: elasticsearch
    template:
      metadata:
        labels:
          app.kubernetes.io/name: elasticsearch
      spec:
        containers:
          - name: elasticsearch
            image: elasticsearch:7.9.3
            env:
              - name: discovery.type
                value: single-node
              - name: path.logs
                value: /var/log/elasticsearch
            volumeMounts:
              - name: logs
                mountPath: /var/log/elasticsearch
              - name: logging-config
                mountPath: /usr/share/elasticsearch/config/log4j2.properties
                subPath: log4j2.properties
                readOnly: true
            ports:
              - name: http
                containerPort: 9200
          - name: logs
            image: alpine:3.12
            command:
              - tail
              - -f
              - /logs/docker-cluster_server.json
            volumeMounts:
              - name: logs
                mountPath: /logs
                readOnly: true
        volumes:
          - name: logging-config
            configMap:
              name: elasticsearch-logging
          - name: logs
            emptyDir: {}
  ```

  > The logging configuration file is a separate `ConfigMap` that's too long to include here.

  Some other use-cases:

  - Reloading ConfigMaps in real-time without requiring pod restarts.
  - Injecting secrets from Hashicorp Vault into your application.
  - Adding a local Redis instance to your application for low-latency in-memory caching.

## initContainers

If we want to run a process that runs to completion in a container.
For example, a process that pulls a code or binary from a repository that will be used by the main web application. That is a task that will be run only once when the pod is first created, or a process that waits from an external service or database to be up before the actual application starts. This is where `initContainers` come in.

```yaml
# pod-init-containers.yml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    name: myapp
spec:
  containers:
  - name: myapp-core
    image: busybox
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    command: ['sh', '-c', 'echo The app is running ! && sleep 3600']
  initContainers:
    - name: init-mysvc
      image: busybox
      command: ['sh', '-c', 'git clone https://github.com/aditya109/web-crawler-nodejs; done;']
```

When a pod is first created the `initContainer` is run, and the process in the `initContainer` must run to a completion before the real container hosting the application starts.

You can configure multiple such `initContainers` as well, like we did for multi-pod containers. In that case each `initContainers` is run **one at a time in sequential order.**

If any of the `initContainers` fail to complete, Kubernetes restarts the Pod repeatedly until the `initContainer` succeeds.

`initContainers` are exactly like regular containers, except:

- `initContainers` always run to completion.
- Each init container must complete successfully before the next one starts.

If a Pod's init container fails, the kubelet repeatedly restarts the init container until it succeeds. However, if the Pod has a `restartPolicy` of Never, and an init container fails during startup of that Pod, Kubernetes treats the overall Pod as failed.

To specify an init container for a Pod, add the `initContainer` field into the Pod specification, as an array of objects of type Container, alongside the app `containers` array. The status of the init containers is returned in `.status.initContainerStatuses` field as an array of the container statuses (similar to the `.status.containerStatuses` field).

### Differences from regular containers

`initContainers` support all the fields and features of app containers, including resource limits, volume, and security settings. However, the resource requests and limits for an init container are handled differently.

Also, `initContainers` do not support `lifecycle`, `livenessProbe`, `readinessProbe` or `startupProbe` because they must run to completion before the Pod can be ready.

If you specify multiple `initContainers` for a Pod, kubelet runs each init container sequentially. Each init container must succeed before the Pod can be ready. When all of the `initContainers` have run to completion, kubelet initializes the application containers for the Pod and runs them as usual.

[Back to Contents ⬆](#Contents)





















