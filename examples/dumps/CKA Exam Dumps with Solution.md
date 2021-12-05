1. List your `PersistentVolume` by `Name`, `Size`.

   ```yaml
   # first let's create PersistentVolumes
   # pv-volume.yaml
   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: talex-pv-volume
     labels:
       type: local
   spec:
     storageClassName: manua, l
     capacity:
       storage: 52Mi
     accessModes:
       - ReadWriteOnce
     hostPath:
       path: "/mnt/data"
   ---
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: ruby-pv-volume
     labels:
       type: local
   spec:
     storageClassName: manual
     capacity:
       storage: 78Mi
     accessModes:
       - ReadWriteOnce
     hostPath:
       path: "/mnt/data"
   ```

   ```sh
   # creating pv
   kubectl create -f pv-volume.yaml
   persistentvolume/talex-pv-volume created
   persistentvolume/ruby-pv-volume created
   ```

   ```sh
   # sorting pv by name
   kubectl get pv --sort-by=.m# Custom Resource Definitions and Custom Resources

   In order to create a new type of resources, we use CRDs.

   When we query about a resource, custom or not, API server first finds it in the `aggragated ` layer, then in `kubernetes native resources`, and last in `API extensions`.

   etadata.name
   NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   ruby-pv-volume    78Mi       RWO            Retain           Available           manual                  2m11s
   talex-pv-volume   52Mi       RWO            Retain           Available           manual                  2m11s
   ```

   ```sh
   # sorting pv by size# Custom Resource Definitions and Custom Resources

   In order to create a new type of resources, we use CRDs.

   When we query about a resource, custom or not, API server first finds it in the `aggragated ` layer, then in `kubernetes native resources`, and last in `API extensions`.

   # Custom Resource Definitions and Custom Resources

   In order to create a new type of resources, we use CRDs.

   When we query about a resource, custom or not, API server first finds it in the `aggragated ` layer, then in `kubernetes native resources`, and last in `API extensions`.


   kubectl get pv --sort-by=.spec.capacity.storage
   NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   talex-pv-volume   52Mi       RWO            Retain           Available           manual                  15s
   ruby-pv-volume    78Mi       RWO            Retain           Available           manual                  15s
   ```

   ```sh
   kubectl get pv --sort-by=.metadata.name,.spec.capacity.storage
   ```

2. All ready nodes, one of them marked as no-schedule, list all the nodes excluding the label no-schedule.

   ```sh
   kubectl taint nodes node1 nodeEffect=unput:NoSchedule
   ```

3. kubectl command failing with errCustom Resource Definitions and Custom Resources

   In order to create a new type of resources, we use CRDs.

   When we query about a resource, custom or not, API server first finds it in the `aggragated` layer, then in `kubernetes native resources`, and last in `API extensions`.

4. or - `The connection to the server localhost:8080 was refused - did you specify the right host or port?`. Fix it.

   - Browse through the `/etcd/kubernetes/manifests/kube-apiserver.yaml`.

5. Create 2 pods -- `mypod1` and `mypod2` , `mypod2` should be scheduled to run anywhere `mypod1` is running.

   - We can label a node.

     ```sh
     kubectl label node node01 area=selected
     ```

     Put both this label under `nodeSelector` field in the manifest of the node `node01`.

   - We can also taint a particular node and add toleration to the two pods.

   - We can use Node Affinity.

   - We can also use PodAffinity.

     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       creationTimestamp: null
       labels:
         run: nginx
       name: mypod1
     spec:
       containers:
         - image: nginx
           name: nginx
           resources: {}
       dnsPolicy: ClusterFirst
       restartPolicy: Always
     status: {}
     ```

     ```yaml
     apiVersion: v1
     kind: Pod
     metadata:
       creationTimestamp: null
       labels:
         run: nginx
       name: mypod2
     spec:
       affinity:
         podAffinity:
           requiredDuringSchedulingIgnoredDuringExecution:
             - labelSelector:
                 matchExpressions:
                   - key: run
                     operator: In
                     values:
                       - nginx
               topologyKey: topology.kubernetes.io/zone
       containers:
         - image: nginx
           name: nginx
           resources: {}
       dnsPolicy: ClusterFirst
       restartPolicy: Always
     status: {}
     ```

6. Questions on `initContainers`:

   1. A container will start only if a file is present.

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: test-pd
      spec:
        containers:
          - image: alpine
            name: test-container
            command:
              [
                "sh",
                "-c",
                "if [! -f /mnt/a.txt]; then exit; else sleep 999999; fi;]",
              ]
            volumeMounts:
              - mountPath: /mnt
                name: cache-volume
        initContainers:
          - name: install
            image: busybox
            command: ["sh", "-c", "FILE=/mnt/a.txt && echo > $FILE"]
            volumeMounts:
              - name: cache-volume
                mountPath: "/mnt"
        volumes:
          - name: cache-volume
            emptyDir: {}
      ```

7. Create a deployment running nginx version 1.12.2 that will run in 2 pods.

   ```sh
   kubectl create deployment nginx-dep --image=nginx:1.12.2 --replicas=2
   ```

   1. Scale this to 4 pods.

      ```sh
      kubectl scale --replicas=4 deployment.apps/nginx-dep
      ```

   2. Scale it back to 2 pods.

      ```sh
      kubectl scale --replicas=2 deployment.apps/nginx-dep
      ```

   3. Upgrade this to 1.13.8

      ```sh
      kubectl set image deployment nginx-dep nginx=nginx:1.13.8
      ```

   4. Check the status of the upgrade.

      ```sh
      kubectl rollout status deployment nginx-dep
      Waiting for deployment "nginx-dep" rollout to finish: 2 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 2 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 2 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 2 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 2 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 3 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 3 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 3 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 3 out of 4 new replicas have been updated...
      Waiting for deployment "nginx-dep" rollout to finish: 1 old replicas are pending termination...
      Waiting for deployment "nginx-dep" rollout to finish: 1 old replicas are pending termination...
      Waiting for deployment "nginx-dep" rollout to finish: 1 old replicas are pending termination...
      Waiting for deployment "nginx-dep" rollout to finish: 3 of 4 updated replicas are available...
      deployment "nginx-dep" successfully rolled out
      ```

   5. How do you do this in a way that you can see history of what happened?

      ```sh
      kubectl rollout history deployment nginx-dep
      deployment.apps/nginx-dep
      REVISION  CHANGE-CAUSE
      1         <none>
      2         <none>
      ```

   6. Undo the upgrade.

      ```sh
      kubectl rollout undo deployment nginx-dep
      deployment.apps/nginx-dep rolled back
      ```

8. Create a pod that has a liveness check.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     labels:
       test: liveness
     name: liveness-exec
   spec:
     containers:
       - name: liveness
         image: k8s.gcr.io/busybox
         args:
           - /bin/sh
           - -c
           - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
         livenessProbe:
           exec:
             command:
               - cat
               - /tmp/healthy
           initialDelaySeconds: 5
           periodSeconds: 5
   ```

9. Create a pod that has a readiness check.

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     labels:
       test: readiness
     name: readiness-exec
   spec:
     containers:
       - name: readiness
         image: k8s.gcr.io/busybox
         args:
           - /bin/sh
           - -c
           - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
         readinessProbe:
           exec:
             command:
               - cat
               - /tmp/healthy
           initialDelaySeconds: 5
           periodSeconds: 5
   ```

10. Create a busybox container without a manifest. Then edit the manifest.

    ```sh
    kubectl run busybox-without-manifest --image=busybox
    kubectl edit pod busybox-without-manifest

    ---OR---
    kubectl get pod busybox-without-manifest -oyaml > busybox-without-manifest.yaml
    vi busybox-without-manifest.yaml
    ```

11. Create a job that runs every 3 minutes and prints out the current time.

    ```yaml
    kind: CronJob
    metadata:
      name: etm
    spec:
      schedule: "*/3 * * * *"
      jobTemplate:
        spec:
          template:
            spec:
              containers:
                - name: every-three-minute
                  image: busybox
                  imagePullPolicy: IfNotPresent
                  command:
                    - /bin/sh
                    - -c
                    - date
              restartPolicy: OnFailure
    ```

12. Create a job that runs 20 times, 5 containers at a time, and prints "Hello parallel world".

    ```yaml
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: q11
    spec:
      completions: 5
      template:
        spec:
          containers:
          - name: bb1
            image: busybox
            command: ["/bin/sh",  "-c", "echo I am job#1"]
          restartPolicy: Never
      backoffLimit: 4
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: q11
    spec:
      completions: 20
      template:
        metadata:
          labels:
            job-name: q11
        spec:
          containers:
          - name: bb1
            image: busybox
            command: ["/bin/sh",  "-c", "echo I am job#1"]
          - name: bb2
            image: busybox
            command: ["/bin/sh",  "-c", "echo I am job#2"]
          - name: bb3
            image: busybox
            command: ["/bin/sh",  "-c", "echo I am job#3"]
          - name: bb4
            image: busybox
            command: ["/bin/sh",  "-c", "echo I am job#4"]
          - name: bb5
            image: busybox
            command: ["/bin/sh",  "-c", "echo I am job#5"]
          restartPolicy: Never
      backoffLimit: 4
    ```

    ```sh
    for i in $(kubectl get pods -l "job-name=q11" | cut -d" " -f1); do if [ $i != "NAME" ]; then kubectl logs $i --all-containers=true; fi; done;
    # each new line ends in a semi colon
    ```

13. Get the list of pod by doing a CURL to the kube-apiserver.

    - ```sh
      kubectl proxy
      curl -X GET http://127.0.0.1:8001/api/v1/namespaces/default/pods
      ```

    -
    -
    -
    -
    - ```sh
      # create a service account
      kubectl create sa podsa

      # create a clusterrole
      kubectl create clusterrole podcr --verb=get --verb=watch --verb=list --resource=pods

      # create a rolebinding
      create rolebinding podsa:podcr --clusterrole=podcr --serviceaccount=default:podsa

      # get url of kube-apiserver endpoint
      kubectl get endpoints
      API_SERVER=192.168.49.2:8443

      # get secret of your sa
      k describe secret podsa-token-n48xb -oyaml

      # base64 decode your ca.crt and store it in ca.crt
      echo LS0tLS1CRUdJ.....tLS0tCg== | base64 -d > ca.crt

      # base64 decode your ca.crt and store it in TOKEN variable
      TOKEN=$(echo LS0tLS1CRUdJ.....tLS0tCg== | base64 -d)

      curl -s GET $API_SERVER/api/v1/namespaces/default/pods --header "Authorization: Bearer $TOKEN" --cacert ca.crt
      ```

14. Deploy a pod with the wrong image name (like --image=nginy) and find the error message.

    ```sh
    kubectl create ns app
    kubectl run pod --name=bad-pod --image=nginy
    kubectl describe pod bad-pod -A
    ```

15. Get logs for a pod which has multiple containers running.

    ```shell
    for i in $(kubectl get pods -l "pod-name=pod213" | cut -d" " -f1); do; if [ $i != "NAME" ]; then; kubectl logs $i  --all-containers=true; fi; done
    ```

16. Deploy nginx with 3 replicas and then expose a port and use port forwarding to talk to a specific port.

    ```bash
    # create basic nginx deployment
    kubectl create deployment nginx-deployment --port=80 --image=nginx

    # scale the deployment
    kubecl scale deployment nginx-deployment --replicas=3

    # expose the deployment
    kubectl expose deployment nginx-deployment --port=80 --name=nginx-svc

    # use port-forwarding to forward traffice
    kubectl port-forward svc nginx-svc :80
    Forwarding from 127.0.0.1:42741 -> 80
    Forwarding from [::1]:42741 -> 80

    # curl the site
    curl 127.0.0.1:42741
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    ```

17. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx.

    ```sh
    k expose deployment nginx-dep --port=80 --target-port=80 --name=nginx-svc --type=LoadBalancer
    ```

18. Get the status of all the master components.

    ```bash
    vagrant@kubemaster:~/spec/question16$ k get componentstatus
    Warning: v1 ComponentStatus is deprecated in v1.19+
    NAME                 STATUS      MESSAGE                                                                                       ERROR
    scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused
    controller-manager   Healthy     ok
    etcd-0               Healthy     {"health":"true","reason":""}
    vagrant@kubemaster:~/spec/question16$ k cluster-info
    Kubernetes control plane is running at https://192.168.56.2:6443
    CoreDNS is running at https://192.168.56.2:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
    ```

19. Create a pod that runs on a given node.
    _Refer to question 4_

20. Create a pod that uses secrets.

    1. Pull secrets from environment variable.

    ```yaml
    # create a secret
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      USER_NAME: YWRtaW4=
      PASSWORD: MWYyZDFlMmU2N2Rm
    ```

    Now create a Secret object.

    ```bash
    k create -f secret.yaml
    ```

    Now create your pod.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: pod-with-secret
      name: pod-with-secret
    spec:
      containers:
        - image: k8s.gcr.io/busybox
          name: pod-with-secret
          command: ["/bin/sh", "-c", "env"]
          envFrom:
            - secretRef:
                name: mysecret
          resources: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
    status: {}
    ```

    2. Pull secrets from a volume.

       ```yaml
       apiVersion: v1
       kind: Pod
       metadata:
         name: secret-test-pod
         labels:
           name: secret-test
       spec:
         volumes:
           - name: secret-volume
             secret:
               secretName: prod-db-secret
         containers:
           - name: ssh-test-container
             image: busybox
             command: ["/bin/sh", "-c", "sleep 99999999"]
             volumeMounts:
               - name: secret-volume
                 readOnly: true
                 mountPath: "/etc/secret-volume"
       ```

       ```bash
       k exec -it pod/secret-test-pod -- /bin/sh
       > ls /etc/secret-volume -l
       total 0
       lrwxrwxrwx    1 root     root            15 Nov 21 15:08 password -> ..data/password
       lrwxrwxrwx    1 root     root            15 Nov 21 15:08 username -> ..data/username
       ```

    3. Dump the secrets out via kubectl to show it worked.

       ```bash
       echo $(k get secret prod-db-secret -o=jsonpath='{range .items[*]}{.data.username}') | base64 -d
       ```

21. Create a static pod and then delete the pod.

    ```bash
    kubectl run nginx-pod --image=nginx --port=80 --dry-run=client -oyaml > nginx-pod.yaml
    sudo mv nginx-pod.yaml /etc/kubernetes/manifests # to create a static pod

    sudo rm -rf nginx-pod.yaml # to delete a static pod
    ```

22. \*Create a pod that do not get IP from the range of allocated CIDR block. Ensure that this is not a static pod.

    ```sh

    ```

23. \*Create a pod that uses a scratch disk.

    1. Change the pod to mount a disk from the host. [Local-PV]

       ```bash
       # create a storage class -- local-sc.yaml
       cat > local-sc.yaml << EOF
       apiVersion: storage.k8s.io/v1
       kind: StorageClass
       metadata:
         creationTimestamp: "2021-11-23T17:04:14Z"
         name: local-storage
         resourceVersion: "53490"
         uid: 4c64dd7a-5f85-47e6-8ad2-9f1ea66557a9
       provisioner: kubernetes.io/no-provisioner
       reclaimPolicy: Delete
       volumeBindingMode: WaitForFirstConsumer
       EOF

       # create the storage class
       kubectl create -f local-sc.yaml

       # create local pv -- local-pv.yaml
       cat > local-pv.yaml << EOF
       apiVersion: v1
       items:
       - apiVersion: v1
         kind: PersistentVolume
         metadata:
           creationTimestamp: "2021-11-23T17:02:50Z"
           finalizers:
           - kubernetes.io/pv-protection
           name: my-local-pv
           resourceVersion: "53374"
           uid: 07b8b57b-b3ca-4842-af71-050261687b75
         spec:
           accessModes:
           - ReadWriteOnce
           capacity:
             storage: 10Mi
           local:
             path: /mnt/disks/vol1
           nodeAffinity:
             required:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname
                   operator: In
                   values:
                   - kubenode01
           persistentVolumeReclaimPolicy: Delete
           storageClassName: local-storage
           volumeMode: Filesystem
         status:
           phase: Available
       kind: List
       metadata:
         resourceVersion: ""
         selfLink: ""
       EOF

       # create local pv
       kubectl create -f local-pv.yaml


       ```

       <https://vocon-it.com/2018/12/20/kubernetes-local-persistent-volumes/#Step_1_Create_StorageClass_with_WaitForFirstConsumer_Binding_Mode>

    2. Change the pod to mount a persistent volume. [hostPath PV]

24. Create a service that manually requires endpoint creation - and create that too.

    For this we create a service without a selector, when such a service is created, the coreresponding endpoint object is not created automatically. We then manually map the service to the network address and port where it's running, by adding an endpoint object manually.

    ```bash
    # create svc.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      ports:
        - protocol: TCP
          port: 80
          targetPort: 9376

    # create my-service service object
    kubectl create -f svc.yaml

    # create ep.yaml
    apiVersion: v1
    kind: Endpoints
    metadata:
      name: my-service
    subsets:
      - addresses:
          - ip: 192.0.2.42
        ports:
          - port: 9376

    # create endpoint object
    kubectl create -f ep.yaml

    # describing my-service endpoint
    k describe endpoints my-service
    Name:         my-service
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>
    Subsets:
      Addresses:          192.0.2.42
      NotReadyAddresses:  <none>
      Ports:
        Name     Port  Protocol
        ----     ----  --------
        <unset>  9376  TCP

    # describing my-service service
    k describe svc my-service
    Name:              my-service
    Namespace:         default
    Labels:            <none>
    Annotations:       <none>
    Selector:          <none>
    Type:              ClusterIP
    IP Family Policy:  SingleStack
    IP Families:       IPv4
    IP:                10.100.100.225
    IPs:               10.100.100.225
    Port:              <unset>  80/TCP
    TargetPort:        9376/TCP
    Endpoints:         192.0.2.42:9376
    Session Affinity:  None
    Events:            <none>
    ```

25. Create a daemon set and change the update strategy to do a rolling update but delaying 30 seconds.

    Use initContainers to cause a delay.

    ```bash
    cat > daemonset.yaml << EOF
    # create a daemon set with OnDelete strategy and image with older version
    # without init container for delay
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: fluentd-elasticsearch
      namespace: default
      labels:
        k8s-app: fluentd-logging
    spec:
      selector:
        matchLabels:
          name: fluentd-elasticsearch
      updateStrategy: OnDelete
      template:
        metadata:
          labels:
            name: fluentd-elasticsearch
        spec:
          tolerations:
          # this toleration is to have the daemonset runnable on master nodes
          # remove it if your masters can't run pods
          - key: node-role.kubernetes.io/master
            operator: Exists
            effect: NoSchedule
          containers:
          - name: fluentd-elasticsearch
            image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
            resources:
              limits:
                memory: 200Mi
              requests:
                cpu: 100m
                memory: 200Mi
            volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
          terminationGracePeriodSeconds: 30
          volumes:
          - name: varlog
            hostPath:
              path: /var/log
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers
      EOF
    # create the daemonset
    kubectl create -f daemonset.yaml

    # edit the daemonset yaml
    # change the strategy and init container for delay
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: fluentd-elasticsearch
      namespace: default
      labels:
        k8s-app: fluentd-logging
    spec:
      selector:
        matchLabels:
          name: fluentd-elasticsearch

      updateStrategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
      template:
        metadata:
          labels:
            name: fluentd-elasticsearch
        spec:
          tolerations:
          # this toleration is to have the daemonset runnable on master nodes
          # remove it if your masters can't run pods
          - key: node-role.kubernetes.io/master
            operator: Exists
            effect: NoSchedule
          initContainers:
          - name: init-myservice
            image: busybox:1.28
            command: ['sh', '-c', "sleep 30"]
          containers:
          - name: fluentd-elasticsearch
            image: quay.io/fluentd_elasticsearch/fluentd:v3.3.0
            resources:
              limits:
                memory: 200Mi
              requests:
                cpu: 100m
                memory: 200Mi
            volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
          terminationGracePeriodSeconds: 30
          volumes:
          - name: varlog
            hostPath:
              path: /var/log
          - name: varlibdockercontainers
            hostPath:
              path: /var/lib/docker/containers

    # apply the changes yaml
    kubectl apply -f daemonset.yaml
    ```

26. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%.

    ```
    desiredReplicas = ceil[currentReplicas * ( currentMetricValue / desiredMetricValue )]
    ```

    For me the autoscaler was not working as the metrics server was not deployed.
    So I deployed the same.

    ```bash
    # deploy metrics server
    wget https://github.com/kubernetes-sigs/metrics-server/releases/download/metrics-server-helm-chart-3.7.0/components.yaml

    # add the following in the command section of the deployment
    # skip if already there

     - args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --kubelet-use-node-status-port
            - --metric-resolution=15s
            - --kubelet-insecure-tls
    # deploy the metrics server manifest
    kubectl create -f components.yaml

    # cat > sample-app.yaml <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: php-apache
    spec:
      selector:
        matchLabels:
          run: php-apache
      replicas: 1
      template:
        metadata:
          labels:
            run: php-apache
        spec:
          containers:
          - name: php-apache
            image: k8s.gcr.io/hpa-example
            ports:
            - containerPort: 80
            resources:
              limits:
                cpu: 500m
              requests:
                cpu: 200m
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: php-apache
      labels:
        run: php-apache
    spec:
      ports:
      - port: 80
      selector:
        run: php-apache

    # we put an autoscaler
    kubectl autoscale deployment php-apache --cpu-percent=50 --min=2 --max=10

    # get pods
    k get hpa
    NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    php-apache   Deployment/php-apache   0%/50%    2         10        2          73s

    # simulate load addition
    kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

    # watch your pods autoscaler
    watch kubectl get pods -A

    ```

27. Create a custom resource definition and display it in the API with cURL.

# Custom Resource Definitions and Custom Resources

    In order to create a new type of resources, we use CRDs.

    When we query about a resource, custom or not, API server first finds it in the `aggragated` layer, then in `kubernetes native resources`, and last in `API extensions`.

    ```bash
    # create crd manifest

    cat > crd.yaml << EOF
    > apiVersion: apiextensions.k8s.io/v1
    > kind: CustomResourceDefinition
    > metadata:
    >   # name must match the spec fields below, and be in the form: <plural>.<group>
    >   name: crontabs.stable.example.com
    > spec:
    >   # group name to use for REST API: /apis/<group>/<version>
    >   group: stable.example.com
    >   # list of versions supported by this CustomResourceDefinition
    >   versions:
    >     - name: v1
    >       # Each version can be enabled/disabled by Served flag.
    >       served: true
    >       # One and only one version must be marked as the storage version.
    >       storage: true
    >       schema:
    >         openAPIV3Schema:
    >           type: object
    >           properties:
    >             spec:
    >               type: object
    >               properties:
    >                 cronSpec:
    >                   type: string
    >                 image:
    >                   type: string
    >                 replicas:
    >                   type: integer
    >   # either Namespaced or Cluster
    >   scope: Namespaced
    >   names:
    >     # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    >     plural: crontabs
    >     # singular name to be used as an alias on the CLI and for display
    >     singular: crontab
    >     # kind is normally the CamelCased singular type. Your resource manifests use this.
    >     kind: CronTab
    >     # shortNames allow shorter string to match your resource on the CLI
    >     shortNames:
    >     - ct
    > EOF

    # register crd api
    kubectl create -f crd.yaml

    # find in using API server
    # start proxy on one session
    kubectl proxy

    # on another session
    curl http://localhost:8001/apis/stable.example.com/v1/crontabs

    # OR use kubectl directly
    kubectl get crontabs
    ```

28. Create a service that references an externalname and test that this works from another pod.

    An **ExternalName Service** is a special case of Service that does not have selectors and uses DNS names instead.

    ```sh
    # lets create an nginx pod
    kubectl run nginx --image=nginx --port=80 --dry-run=client -oyaml > nginx.yaml

    # create nginx pod
    kubectl create -f nginx.yaml

    # create an external name service
    cat > nginx-svc.yaml << EOF
    apiVersion: v1
    kind: Service
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
      name: nginx
    spec:
      ports:
      - port: 80
        protocol: TCP
        targetPort: 80
      selector:
        run: nginx
      type: ExternalName
      externalName: mycustomnginx.cross.system.io
    status:
      loadBalancer: {}
    EOF

    # create service
    kubectl create -f nginx-svc.yaml

    # create a test-pod now and hit the above service
    cat > test-pod.yaml << EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: test-pod
      labels:
        app: test
    spec:
      containers:
      - name: test-container
        image: nginx
        command: ['sh', '-c', 'http://mycustomnginx.cross.system.io:80']
    EOF

    # create a test pod
    kubectl create -d test-pod.yaml

    # check logs
    kubectl logs -f test-pod

    curl http://mycustomnginx.cross.system.io
    <html>
    <head><title>301 Moved Permanently</title></head>
    <body bgcolor="white">
    <center><h1>301 Moved Permanently</h1></center>
    <hr><center>nginx/1.14.2</center>
    </body>
    </html>
    ```

29. Create a pod that runs all processes as user 1000.

    ```sh
    # let's create a pod with user
    cat > busy-box-pod.yaml << EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: security-context-demo
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      volumes:
      - name: sec-ctx-vol
        emptyDir: {}
      containers:
      - name: sec-ctx-demo
        image: busybox
        command: [ "sh", "-c", "sleep 1h" ]
        volumeMounts:
        - name: sec-ctx-vol
          mountPath: /data/demo
        securityContext:
          allowPrivilegeEscalation: false
    EOF


    # now create pod
    kubectl create -f busy-box-pod.yaml
    ```

30. Write an ingress rule that redirects calls to `/foo` to one service and to `/bar` to another.

    ```sh
    # first we install the nginx-ingress-controller

    # then we create two deployments and their respective services
    kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
    kubectl expose deployment web --type=NodePort --port=8080


    kubectl create deployment web2 --image=gcr.io/google-samples/hello-app:2.0
    kubectl expose deployment web2 --port=8080 --type=NodePort

    # create an ingress now
    cat > ingress.yaml <<EOF
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: example-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /$1
    spec:
      rules:
        - host: hello-world.info
          http:
            paths:
              - path: /foo
                pathType: Prefix
                backend:
                  service:
                    name: web
                    port:
                      number: 8080
              - path: /bar
                pathType: Prefix
                backend:
                  service:
                    name: web2
                    port:
                      number: 8080
    EOF

    # create the ingress and wait for it to get ADDRESS field
    kubectl get ingress
    NAME              CLASS    HOSTS              ADDRESS        PORTS   AGE
    example-ingress   <none>   hello-world.info   172.17.0.15    80      38s

    # if you are using minikube, get the external IP of minikube using:
    minikube ip
    192.168.49.2

    # pick this IP and add it to the list of `/etc/hosts`
    EXTERNAL_IP   192.168.49.2

    # curl the hostname
    curl hello-world.info/foo
    Hello, world!
    Version: 1.0.0
    Hostname: web-79d88c97d6-t4qn7

    curl hello-world.info/bar
    Hello, world!
    Version: 2.0.0
    Hostname: web2-5d47994f45-2qc9g
    ```

31. Write a service that exposes nginx on a nodeport.

    1. Change it to use a cluster port.
       _Edit the target port (I think_ 🤔 _)_

    2. Scale the deployment.
    3. Change it to use an external IP. (can be done)
    4. Change it to use a load balancer. (can be done)

    <https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer>

    ```sh
    # create nginx deployment
    kubectl create deployment nginx-dep --image=nginx --port=80
    kubectl expose deployment nginx-dep --port=80 --type=NodePort
    ```

    A ClusterIP exposes the following:

    `spec.clusterIp:spec.ports[*].port`
    You can only access this service while inside the cluster. It is accessible from its `spec.clusterIp` port. If a `spec.ports[*].targetPort` is set it will route from the port to the targetPort. The CLUSTER-IP you get when calling kubectl get services is the IP assigned to this service within the cluster internally.

    A NodePort exposes the following:

    - `<NodeIP>:spec.ports[*].nodePort`
    - `spec.clusterIp:spec.ports[*].port`

    If you access this service on a nodePort from the node's external IP, it will route the request to `spec.clusterIp:spec.ports[*].port`, which will in turn route it to your `spec.ports[*].targetPort`, if set. This service can also be accessed in the same way as ClusterIP.

    Your NodeIPs are the external IP addresses of the nodes. You cannot access your service from `spec.clusterIp:spec.ports[*].nodePort`.

    A LoadBalancer exposes the following:

    - `spec.loadBalancerIp:spec.ports[*].port`
    - `<NodeIP>:spec.ports[*].nodePort`
    - `spec.clusterIp:spec.ports[*].port`

    You can access this service from your load balancer's IP address, which routes your request to a nodePort, which in turn routes the request to the clusterIP port. You can access this service as you would a NodePort or a ClusterIP service as well.

32. Deploy nginx with 3 replicas and then expose a port and use port forwarding to talk to a specific port.

    ```sh
    # create an nginx deployment
    kubectl create deployment nginx-dep --image=nginx --port=80

    # use port forwarding to ping the pod
    kubectl port-forward pod/nginx-7848d4b86f-69z66 8080:80

    # now try hitting the port forwarding the pod
    curl http://127.0.0.1:8080
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    <style>
    html { color-scheme: light dark; }
    body { width: 35em; margin: 0 auto;
    font-family: Tahoma, Verdana, Arial, sans-serif; }
    </style>
    </head>
    <body>
    <h1>Welcome to nginx!</h1>
    <p>If you see this page, the nginx web server is successfully installed and
    working. Further configuration is required.</p>

    <p>For online documentation and support please refer to
    <a href="http://nginx.org/">nginx.org</a>.<br/>
    Commercial support is available at
    <a href="http://nginx.com/">nginx.com</a>.</p>

    <p><em>Thank you for using nginx.</em></p>
    </body>
    </html>
    ```

33. Get logs for Kubernetes master components.

    ```sh
    # To get detailed information about the overall health of your cluster, you can run:
    kubectl cluster-info dump
    ```

34. Get logs for Kubelet. (simple)

35. Backup an etcd cluster. (simple)

36. List the members of an etcd cluster. (simple)

37. Find the health of etcd. (simple)

38. Create a namespace [Important]

    1. Run a pod in the new namespace.
    2. Put memory limits on the namespace.
    3. Limit pods to 2 persistent volumes in this namespace

    ```sh
    # creating a new namespace
    kubectl create ns ns38

    # creating a new pod in ns38 namespace
    kubectl run nginx -n ns38 --port=80

    # create a limit range manifest
    cat > lr.yaml <<EOF
    apiVersion: v1
    kind: LimitRange
    metadata:
      name: mem-limit-range
      namespace: ns38
    spec:
      limits:
      - default:
          memory: 512Mi
        defaultRequest:
          memory: 256Mi
        type: Container
    EOF

    # apply manifest
    k apply -f lr.yaml
    ```

    https://devspace.cloud/cloud/docs/admin/spaces/limits-isolation/namespace-limits

39. Create a networking policy such that only pods with the label access=granted can talk to it.

    1. Create an nginx pod and attach this policy to it.
    2. Create a busybox pod and attempt to talk to nginx - should be blocked.
    3. Attach the label to busybox and try again - should be allowed.

40. Create a multi containers of `nginx`, `redis` and `consul`.

    ```yaml
    kind: Pod
    metadata:
      creationTimestamp: null
      labels:
        run: pod-m
      name: pod-m
    spec:
      containers:
        - image: nginx
          name: nginx
          resources: {}
        - name: redis
          image: redis
          volumeMounts:
            - name: redis-storage
              mountPath: /data/redis
        - name: consul
          image: consul
      volumes:
        - name: redis-storage
          emptyDir: {}
      dnsPolicy: ClusterFirst
      restartPolicy: Always
    status: {}
    ```

41. Troubleshooting not ready state node.

42. Add missing worker node -- TLS bootstrapping.

43. Set up a Kubernetes cluster from scratch by using Kubeadm. [done]

44. Create Redis pod without using PV.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: redis
    spec:
      containers:
        - name: redis
          image: redis
          volumeMounts:
            - name: redis-storage
              mountPath: /data/redis
      volumes:
        - name: redis-storage
          emptyDir: {}
    ```

45. Creating PVolume with host path.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: task-pv-volume
      labels:
        type: local
    spec:
      storageClassName: manual
      capacity:
        storage: 10Gi
      accessModes:
        - ReadWriteOnce
      hostPath:
        path: "/mnt/data"
    ```

46. Create pods,service in particular namespace, list all services in particular namespace.

    ```sh
    kubectl get svc -n NAMESPACE_NAME
    ```

47. Create nginx deployment nginx-random expose it; then create another pod busybox and do the following:

    1. Dnlookup service
    2. Dnslookup pod

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx
    spec:
      selector:
        matchLabels:
          run: my-nginx
      replicas: 2
      template:
        metadata:
          labels:
            run: my-nginx
        spec:
          containers:
            - name: my-nginx
              image: nginx
              ports:
                - containerPort: 80
    ```

    ```sh
    kubectl create -f my-nginx.yaml
    kubectl expose deployment/my-nginx

    kubectl run curl --image=radial/busyboxplus:curl -i --tty
    kubectl exec -it curl -- sh
    nslookup my-nginx
    nslookup my-nginx-5b56ccd65f-9nrh6
    ```

48. Expose a service to Nodeport. Edit the yaml.

    ```sh
    kubectl expose service nginx --port=443 --target-port=8443 --name=nginx-https --type=NodePort --dry-run=client -oyaml > nginx-np.yaml
    ```

49. Create a pod that by passes kube-scheduler. Ensure that this is not a static pod.
    - Add a `nodeSelector` field
    - Add `schedulerName` field

<https://medium.com/@sensri108/practice-examples-dumps-tips-for-cka-ckad-certified-kubernetes-administrator-exam-by-cncf-4826233ccc27>

<https://devops.digit.org/hiring-devops/devops-hiring/exercise-2>

<https://github.com/Kasunmadura/k8s/blob/master/cka-exam/README.md>
