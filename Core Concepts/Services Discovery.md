# Service Discovery

Let's say we have the following service running in the namespace `restapi`.

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: restapi
  name: restapi
  namespace: restapi
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: restapi
status:
  loadBalancer: {}
```

We want to access this service in our new `nginx` pod.

```sh
kubectl create pod --image=nginx --namespace=restapi
```

There are two ways of accessing the service:

1. *Using environment variable*
   In the environment variable method, the service to be accessed is supposed to have two conditions:
   a.	The service should be created before accessing pod creation.
   b.	The pod should be in same namespace.

   Let's try to `exec` into the pod.

```sh
kubectl -n restapi exec -it nginx -- bash
root@nginx:/# env
KUBERNETES_SERVICE_PORT_HTTPS=443
RESTAPI_SERVICE_HOST=10.97.181.20			👈
KUBERNETES_SERVICE_PORT=443
RESTAPI_PORT=tcp://10.97.181.20:8080		👈
HOSTNAME=nginx
RESTAPI_SERVICE_PORT=8080					👈
PWD=/
PKG_RELEASE=1~buster
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
RESTAPI_PORT_8080_TCP=tcp://10.97.181.20:8080
RESTAPI_PORT_8080_TCP_PORT=8080
NJS_VERSION=0.6.2
TERM=xterm
SHLVL=1
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
RESTAPI_PORT_8080_TCP_ADDR=10.97.181.20
KUBERNETES_SERVICE_HOST=10.96.0.1
RESTAPI_PORT_8080_TCP_PROTO=tcp
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
NGINX_VERSION=1.21.3
_=/usr/bin/env
```

> As you can see, it matches the service `restapi` IP, if we `get` the service from the corresponding namespace.
>
> ```sh
> >: kubectl get svc -owide -n restapi
> NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE    SELECTOR
> restapi   ClusterIP   10.97.181.20   <none>        8080/TCP   141m   app=restapi
> ```

```sh
root@nginx:/# curl http://10.97.181.20:8080/apis/v1/books
[{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"}]
root@nginx:/# curl http://10.97.181.20:8080/apis/v1/books
[{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"}]
```

2. *Using DNS Method*

   `svc_name.namespace.svc.cluster.local`

   For this you don't need any restrictions foreither the service or pod.

   ```sh
   root@nginx:/# curl restapi.restapi.svc.cluster.local:8080/apis/v1/books
   [{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"}]
   ```

## Types of services

There are 2 types of services:

1. ClusterIP (*normally used, facilitates interpod communication*)

2. NodePort (*for exposing pod via its own private IP to external world*)
   Let's try to expose our exisiting application using NodePort service.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: null
     labels:
       app: restapi
     name: restapi-np-type
     namespace: restapi
   spec:
     ports:
     - port: 8080
       protocol: TCP
       targetPort: 8080
     selector:
       app: restapi
     type: NodePort
   status:
     loadBalancer: {}
   ```

   If we `create` this NodePort service, and we try to  `describe` it:

   ```sh
   >: kubectl describe services -n restapi
   Name:              restapi
   Namespace:         restapi
   Labels:            app=restapi
   Annotations:       <none>
   Selector:          app=restapi
   Type:              ClusterIP
   IP Family Policy:  SingleStack
   IP Families:       IPv4
   IP:                10.97.181.20
   IPs:               10.97.181.20
   Port:              <unset>  8080/TCP
   TargetPort:        8080/TCP
   Endpoints:         172.17.0.2:8080
   Session Affinity:  None
   Events:            <none>
   
   
   Name:                     restapi-np-type
   Namespace:                restapi
   Labels:                   app=restapi
   Annotations:              <none>
   Selector:                 app=restapi
   Type:                     NodePort
   IP Family Policy:         SingleStack
   IP Families:              IPv4
   IP:                       10.105.175.45
   IPs:                      10.105.175.45
   Port:                     <unset>  8080/TCP
   TargetPort:               8080/TCP
   NodePort:                 <unset>  32628/TCP
   Endpoints:                172.17.0.2:8080
   Session Affinity:         None
   External Traffic Policy:  Cluster
   Events:                   <none>
   ```

   This application will now be accessible to you on your respective node's IP.
   `NODE_IP:NODEPORT_OF_NPSERVICE`.

   For me,  `-owide`/`describe` output for nodes was:

   ```sh
   >: kubectl describe nodes 
   Name:               minikube
   Roles:              control-plane,master
   Labels:             beta.kubernetes.io/arch=amd64
   Addresses:
     InternalIP:  192.168.49.2 			👈
     Hostname:    minikube
   ...
   ```

   

   ```sh
   >: curl http://192.168.49.2:32628/apis/v1/books
   [{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"},{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"}]
   ```

   

3. LoadBalancer (*cloud controller manager creates an LB*)
   Let's try to expose our exisiting application using LoadBalancer service.

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: null
     labels:
       app: restapi
     name: restapi-lb-type
     namespace: restapi
   spec:
     ports:
     - port: 8080
       protocol: TCP
       targetPort: 8080
     selector:
       app: restapi
     type: LoadBalancer
   status:
     loadBalancer: {}
   ```

   Normally if you just `create` this service if you were using a cloud provider, it would automatically work.

   But if you are using `minikube` like I am, run the following command on a separate terminal,

   ```sh
   minikube tunnel
   ```

   Then when you create you show see something like this,

   ```sh
   >: kubectl describe -n restapi svc restapi-lb-type 
   Name:                     restapi-lb-type
   Namespace:                restapi
   Labels:                   app=restapi
   Annotations:              <none>
   Selector:                 app=restapi
   Type:                     LoadBalancer
   IP Family Policy:         SingleStack
   IP Families:              IPv4
   IP:                       10.110.70.159
   IPs:                      10.110.70.159
   LoadBalancer Ingress:     10.110.70.159
   Port:                     <unset>  8080/TCP
   TargetPort:               8080/TCP
   NodePort:                 <unset>  31601/TCP
   Endpoints:                172.17.0.2:8080
   Session Affinity:         None
   External Traffic Policy:  Cluster
   Events:                   <none>
   ```

   Now if you EXTERNAL_IP:TARGET_PORT, that should give you the result.

   ```bash
   >: curl http://10.110.70.159:8080/apis/v1/books
   [{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"},{"Id":"1","Name":"The Everything Store","Isbn":"ISBN-1"}]
   ```

4. ExternalName (*resolve a particular service to given DNS name*)

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     creationTimestamp: null
     labels:
       app: restapi
     name: restapi-en-type
     namespace: restapi
   spec:
     ports:
     - port: 8080
       protocol: TCP
       targetPort: 8080
     selector:
       app: restapi
   status:
     loadBalancer: {}
   ```

   



