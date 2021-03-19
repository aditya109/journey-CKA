# Application Lifecycle Management

## Rolling Updates and Rollbacks in Deployments

### Rollout and Versioning

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Application Lifecycle Management/assets/rollouts.svg)

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
  ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Application Lifecycle Management/assets/st1.svg)
  The problem with this strategy is if something goes wrong in the update process, we neither have any running instances to cater to existing users, nor do we have the new instances, (as the something could have gone wrong, or amidst update process), there is no running instance for catering users, which is really **BFB** (Bad for Business).

- Bring alternative instances of our application down, while keeping the left-out instances running on older version, and then bring new instances to replace downed instances. This is called **Rolling Update** strategy.
  ![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Application Lifecycle Management/assets/st2.svg)

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

Still if you do `docker ps -a`, you would see the container details.

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

As you can see `bash` is a shell not a process, it listens to input on the `stdout`, but if it doesn't find any it exits out.

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

## Scale Applications

## Self-Healing Application



