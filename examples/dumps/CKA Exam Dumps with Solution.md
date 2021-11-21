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
   kubectl get pv --sort-by=.metadata.name
   NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   ruby-pv-volume    78Mi       RWO            Retain           Available           manual                  2m11s
   talex-pv-volume   52Mi       RWO            Retain           Available           manual                  2m11s
   ```

   ```sh
   # sorting pv by size
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

3. kubectl command failing with error - `The connection to the server localhost:8080 was refused - did you specify the right host or port?`. Fix it.

   - Browse through the `/etcd/kubernetes/manifests/kube-apiserver.yaml`.

4. Create 2 pods -- `mypod1` and `mypod2` , `mypod2` should be scheduled to run anywhere `mypod1` is running.

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

5. Questions on `initContainers`:

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

6. Create a deployment running nginx version 1.12.2 that will run in 2 pods.

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

7. Create a pod that has a liveness check.
   
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
   
8. Create a pod that has a readiness check.

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

9. Create a busybox container without a manifest. Then edit the manifest.

   ```sh
   kubectl run busybox-without-manifest --image=busybox
   kubectl edit pod busybox-without-manifest

   ---OR---
   kubectl get pod busybox-without-manifest -oyaml > busybox-without-manifest.yaml
   vi busybox-without-manifest.yaml
   ```

10. Create a job that runs every 3 minutes and prints out the current time.

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

12. Get the list of pod by doing a CURL to the kube-apiserver.

    - ```sh
      kubectl proxy
      curl -X GET http://127.0.0.1:8001/api/v1/namespaces/default/pods
      ```
      
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
    
13. Deploy a pod with the wrong image name (like --image=nginy) and find the error message.

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

17. Get the status of all the master components.

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
    *Refer to question 4*

19. Create a pod that uses secrets.
    
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
           command: [ "/bin/sh", "-c", "sleep 99999999" ]
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
    
20. Create a static pod and then delete the pod.

    ```bash
    ```

    

21. Create a pod that do not get IP from the range of allocated CIDR block. Ensure that this is not a static pod.

22. Create a service that uses a scratch disk.
    1. Change the service to mount a disk from the host. [Local-PV]
    2. Change the service to mount a persistent volume. [hostPath PV]
    
23. Create a service that manually requires endpoint creation - and create that too.

24. Create a daemon set and change the update strategy to do a rolling update but delaying 30 seconds.

25. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%.

26. Create a custom resource definition and display it in the API with cURL.

27. Create a service that references an externalname and test that this works from another pod.

28. Create a pod that runs all processes as user 1000.

29. Write an ingress rule that redirects calls to `/foo` to one service and to `/bar` to another.

30. Write a service that exposes nginx on a nodeport.
    1. Change it to use a cluster port.
    2. Scale the service.
    3. Change it to use an external IP.
    4. Change it to use a load balancer.
    
31. Deploy nginx with 3 replicas and then expose a port and use port forwarding to talk to a specific port.

32. Get logs for Kubernetes master components.

33. Get logs for Kubelet.

34. Backup an etcd cluster.

35. List the members of an etcd cluster.

36. Find the health of etcd.

37. Create a namespace [Important]

    1. Run a pod in the new namespace.
    2. Put memory limits on the namespace.
    3. Limit pods to 2 persistent volumes in this namespace

38. Create a networking policy such that only pods with the label access=granted can talk to it.

    1. Create an nginx pod and attach this policy to it.
    2. Create a busybox pod and attempt to talk to nginx - should be blocked.
    3. Attach the label to busybox and try again - should be allowed.

39. Create a multi containers of `nginx`, `redis` and `consul`.

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

40. Troubleshooting not ready state node.

41. Add missing worker node -- TLS bootstrapping.

42. Set up a Kubernetes cluster from scratch by using Kubeadm.

43. Create Redis pod without using PV.

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

44. Creating PVolume with host path.

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

45. Create pods,service in particular namespace, list all services in particular namespace.

    ```sh
    kubectl get svc -n NAMESPACE_NAME
    ```

46. Create nginx deployment nginx-random expose it; then create another pod busybox and do the following:

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

47. Expose a service to Nodeport. Edit the yaml.

    ```sh
    kubectl expose service nginx --port=443 --target-port=8443 --name=nginx-https --type=NodePort --dry-run=client -oyaml > nginx-np.yaml
    ```

48. Create a pod that by passes kube-scheduler. Ensure that this is not a static pod.
    - Add a `nodeSelector` field
    - Add `schedulerName` field

https://medium.com/@sensri108/practice-examples-dumps-tips-for-cka-ckad-certified-kubernetes-administrator-exam-by-cncf-4826233ccc27

https://devops.digit.org/hiring-devops/devops-hiring/exercise-2

https://github.com/Kasunmadura/k8s/blob/master/cka-exam/README.md
