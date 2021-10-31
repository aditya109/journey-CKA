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
2. NodePort ()
3. LoadBalancer
4. ExternalName



