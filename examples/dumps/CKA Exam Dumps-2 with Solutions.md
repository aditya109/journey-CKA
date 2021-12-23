# CKA Exam Dumps-2

1. Create a node that has a SSD and label it as such. ✅
   
   ```sh
   kubectl label node kubenode01 disktype=ssd
   kubectl label node kubenode01 disktype- # to delete the above label
   
2. Create a pod that is only scheduled on SSD nodes. ✅

   ```sh
   cat pod.yaml >> EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: nginx
     labels:
       env: test
   spec:
     containers:
     - name: nginx
       image: nginx
       imagePullPolicy: IfNotPresent
     nodeSelector:
       disktype: ssd
   EOF
   kubectl create -f pod.yaml
   ```

3. Create 2 pod definitions: the second pod should be scheduled to run anywhere the first pod is running - 2nd pod runs alongside the first pod. ✅

   ```sh
   k run nginx --image=nginx -l=pos=first --dry-run=client -oyaml > first-pod.yaml
   # let us make it so that this pod gets scheduled on a particular node so we will add a node selector as well
   cat first-pod.yaml
   apiVersion: v1
   kind: Pod
   metadata:
     creationTimestamp: null
     labels:
       pos: first
     name: nginx
   spec:
     containers:
     - image: nginx
       name: nginx
       resources: {}
     dnsPolicy: ClusterFirst
     restartPolicy: Always
     nodeSelector:
       kubernetes.io/hostname: kubenode01
   status: {}
   # let's create this pod
   kubectl create -f first-pod.yaml
   
   # to make the second co-existing pod
   cat second-pod.yaml >> EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: with-pod-affinity
   spec:
     affinity:
       podAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
         - labelSelector:
             matchExpressions:
             - key: pos 
               operator: In
               values:
               - first
           topologyKey: kubernetes.io/hostname 
     containers:
     - name: with-pod-affinity
       image: k8s.gcr.io/pause:2.0
   EOF
   kubectl create -f second-pod.yaml
   ```

   

4. Create a deployment running nginx version 1.12.2 that will run in 2 pods. ✅
   - Scale this to 4 pods. ✅
   - Scale it back to 2 pods. ✅
   - Upgrade this to 1.13.8. ✅
   - Check the status of the upgrade. ✅
   - How do you do this in a way that you can see the history of what happened? ✅
   - Undo the upgrade. ✅
   
   ```sh
   kubectl create deployment my-dep --replicas=2 --image=nginx:1.12.2 --dry-run=client -oyaml > dep.yaml
   kubectl scale --replicas=4 -f dep.yaml 
   kubectl scale --replicas=2 -f dep.yaml 
   kubectl set image deployments/my-dep nginx:1.12.2=nginx:1.13.8  
   kubectl rollout status deployments/my-dep
   kubectl rollout history deployments/my-dep
   kubectl rollout undo deployments/my-dep
   ```
   
5. ***Create a pod that uses a scratch disk.**
   
   ```sh
   cat > pod1.yaml << EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: myvolumes-pod
   spec:
     containers:
     - image: alpine
       imagePullPolicy: IfNotPresent
       name: myvolumes-container
       command: [    'sh', '-c', 'echo The Bench Container 1 is Running ; sleep 3600']
       volumeMounts:
       - mountPath: /demo
         name: demo-volume
     volumes:
     - name: demo-volume
       emptyDir: {}
   ```
   
   > Here, a directory is created in the container itself, called `demo`. Whatever is created over there, exists as long as container exists. 
   
   ```sh
   cat > pod2.yaml << EOF
   apiVersion: v1
   kind: Pod
   metadata:
     name: myvolumes-pod
   spec:
     containers:
     - image: alpine
       imagePullPolicy: IfNotPresent
       name: myvolumes-container-1
       command: ['sh', '-c', 'echo The Bench Container 1 is Running ; sleep 3600']
       volumeMounts:
       - mountPath: /demo1
         name: demo-volume
     - image: alpine
       imagePullPolicy: IfNotPresent
       name: myvolumes-container-2
       command: ['sh', '-c', 'echo The Bench Container 2 is Running ; sleep 3600']
       volumeMounts:
       - mountPath: /demo2
         name: demo-volume
     - image: alpine
       imagePullPolicy: IfNotPresent
       name: myvolumes-container-3
       command: ['sh', '-c', 'echo The Bench Container 3 is Running ; sleep 3600']
       volumeMounts:
       - mountPath: /demo3
         name: demo-volume
     volumes:
     - name: demo-volume
       emptyDir: {}
   ```
   
   > Here, all the three containers share the `demo` folder.
   
   For making a pod mount a `hostPath` volume.
   
   ```sh
   apiVersion: v1
   kind: Pod
   metadata:
     name: test-webserver
   spec:
     containers:
     - name: test-webserver
       image: k8s.gcr.io/test-webserver:latest
       volumeMounts:
       - mountPath: /var/local/aaa
         name: mydir
       - mountPath: /var/local/aaa/1.txt
         name: myfile
     volumes:
     - name: mydir
       hostPath:
         # Ensure the file directory is created.
         path: /var/local/aaa
         type: DirectoryOrCreate
     - name: myfile
       hostPath:
         path: /var/local/aaa/1.txt
         type: FileOrCreate
   ```
   
   
   
   - *****Change the pod to mount a disk from the host.
   
     
   
   - *****Change the pod to mount a persistent volume.
   
   Reference link: https://www.alibabacloud.com/blog/kubernetes-volume-basics-emptydir-and-persistentvolume_594834
   
6. Create a pod that has a liveness check. ✅

   ```sh
   cat > pod-with-liveness.yaml << EOF
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
   EOF
   
   kubectl create -f pod-with-liveness.yaml
   kubectl describe pod/liveness-exec
   ....
   .....
   Events:
     Type     Reason     Age   From               Message
     ----     ------     ----  ----               -------
     Normal   Scheduled  43s   default-scheduler  Successfully assigned default/liveness-exec to kubenode02
     Normal   Pulling    43s   kubelet            Pulling image "k8s.gcr.io/busybox"
     Normal   Pulled     38s   kubelet            Successfully pulled image "k8s.gcr.io/busybox" in 4.664750908s
     Normal   Created    38s   kubelet            Created container liveness
     Normal   Started    38s   kubelet            Started container liveness
     Warning  Unhealthy  4s    kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
   ```

7. Create a service that manually requires endpoint creation - and create that too. ✅

   ```bash
   cat > service.yaml << EOF
   > apiVersion: v1
   > kind: Service
   > metadata:
   >   name: my-service
   > spec:
   >   ports:
   >     - protocol: TCP
   >       port: 80
   >       targetPort: 9376
   > EOF
   kubectl create -f service.yaml 
   cat > endpoint.yaml << EOF
   > apiVersion: v1
   > kind: Endpoints
   > metadata:
   >   name: my-service
   > subsets:
   >   - addresses:
   >       - ip: 192.0.2.42
   >     ports:
   >       - port: 9376
   > EOF
   kubectl create -f endpoint.yaml

8. Create a daemon set. ✅
   - Change the update strategy to do a rolling update but delaying 5nan seconds between pod updates. ✅
   
   ```bash
   cat > daemonset.yaml << EOF
   apiVersion: apps/v1
   kind: DaemonSet
   metadata:
     name: fluentd-elasticsearch
     labels:
       k8s-app: fluentd-logging
   spec:
     selector:
       matchLabels:
         name: fluentd-elasticsearch
     updateStrategy:
       type: OnDelete
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
           effect: NoSchedule
         containers:
         - name: fluentd-elasticsearch
           image: quay.io/fluentd_elasticsearch/fluentd:v3.1.3
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
   kubectl create -f daemonset.yaml
   ```
   
9. Create a static pod. ✅

   ```bash
   kubectl run nginx-static-pod --image=nginx --dry-run=client -oyaml > pod-nginx.yaml
   ps aux | grep kubelet | grep config
   sudo cp pod-nginx.yaml /etc/kubernetes/manifests
   ```

10. Create a busybox container without a manifest. Then edit the manifest. ✅

    ```sh
    kubectl run busybox-pod --image=busybox
    kubectl edit pod/busybox-pod
    ```

11. Create a pod that uses secrets. ✅
    - Pull secrets from environment variables. ✅
    - Pull secrets from a volume.✅
    - Dump the secrets out via kubectl to show it worked.
    
    ```bash
    kubectl create secret generic s1 --from-literal=logname=$LOGNAME --from-literal=name=ADITYA --dry-run=client -oyaml > secret.yaml 
    cat > pod.yaml << EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: mypod
        image: nginx
        env:
          - name: LOGNAME
            valueFrom:
              secretKeyRef:
                name: s1
                key: logname
          - name: NAME
            valueFrom:
              secretKeyRef:
                name: s1
                key: name
    
    kubectl create -f pod.yaml
    
    =================================================
    cat > pod2.yaml << EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
    spec:
      containers:
      - name: mypod
        image: nginx
        volumeMounts:
        - name: foo
          mountPath: "/etc/foo"
          readOnly: true
      volumes:
      - name: foo
        secret:
          secretName: s1
    kubectl create -f pod2.yaml
    =================================
    k get secret s1 -o jsonpath="{.data.logname}{'\n'}" | base64 -d && k1 -o jsonpath="{.data.name}" | base64 -d 
    ```
    
12. Create a job that runs every 3 minutes and prints out the current time.

    ```sh
    cat > cronjob.yaml << EOF
    apiVersion: batch/v1
    kind: CronJob
    metadata:
      name: hello
    spec:
      schedule: "*/3 * * * *"
      jobTemplate:
        spec:
          template:
            spec:
              containers:
              - name: hello
                image: busybox
                imagePullPolicy: IfNotPresent
                command:
                - /bin/sh
                - -c
                - date; echo hello ! I am working
              restartPolicy: OnFailure
    # to check logs, 
    kubectl get jobs
    hello-27337510   1/1           2s         2m6s
    hello-27337511   1/1           2s         66s
    hello-27337512   1/1           2s         6s
    # use these as selectors to get logs
    pods=$(kubectl get pods --selector=job-name=hello-27337511 --output=jsonpath={.items[*].metadata.name})
    kubectl logs $pods
    ```

13. Create a job that runs 20 times, 5 containers at a time, and prints "Hello parallel world".

    ```bash
    cat > job.yaml << EOF
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: hello-job
    spec:
      suspend: false
      parallelism: 5
      completions: 20
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            command: ["/bin/sh",  "-c", "echo Hello parallel world"]
          restartPolicy: OnFailure
    EOF
    kubectl create -f job.yaml 
    k logs -f -l=job-name=hello-job --max-log-requests=30
    ```

14. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx.

15. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%.

16. Create a custom resource definition.
    - Display it in the API with curl.
    
    ```bash
    cat > crd.yaml << EOF
    apiVersion: apiextensions.k8s.io/v1
    kind: CustomResourceDefinition
    metadata:
      # name must match the spec fields below, and be in the form: <plural>.<group>
      name: kratos.stable.example.com
    spec:
      # group name to use for REST API: /apis/<group>/<version>
      group: stable.example.com
      # list of versions supported by this CustomResourceDefinition
      versions:
        - name: v1
          # Each version can be enabled/disabled by Served flag.
          served: true
          # One and only one version must be marked as the storage version.
          storage: true
          schema:
            openAPIV3Schema:
              type: object
              properties:
                spec:
                  type: object
                  properties:
                    cronSpec:
                      type: string
                    image:
                      type: string
                    replicas:
                      type: integer
      # either Namespaced or Cluster
      scope: Namespaced
      names:
        # plural name to be used in the URL: /apis/<group>/<version>/<plural>
        plural: kratos
        # singular name to be used as an alias on the CLI and for display
        singular: krato
        # kind is normally the CamelCased singular type. Your resource manifests use this.
        kind: Krato
        # shortNames allow shorter string to match your resource on the CLI
        shortNames:
        - kt
    curl http://127.0.0.1:8001/apis/stable.example.com/v1/kratos
    ```
    
    
    
17. Create a networking policy such that only pods with the label access=granted can talk to it.
    - Create an nginx pod and attach this policy to it.
    - Create a busybox pod and attempt to talk to nginx - should be blocked
    - Attach the label to busybox and try again - should be allowed
    
18. Create a service that references an external name.
    - Test that this works from another pod
    
19. Create a pod that runs all processes as user 1000.

20. Create a namespace
    - Run a pod in the new namespace
    - Put memory limits on the namespace
    - Limit pods to 2 persistent volumes in this namespace
    
21. Write an ingress rule that redirects calls to /foo to one service and to /bar to another

22. Write a service that exposes nginx on a nodeport
    - Change it to use a cluster port
    - Scale the service
    - Change it to use an external IP
    - Change it to use a load balancer
    
23. Deploy nginx with 3 replicas and then expose a port
    - Use port forwarding to talk to a specific port
    
24. Make an API call using CURL and proper certs

25. Upgrade a cluster with kubeadm.

26. Get logs for a pod.

27. Deploy a pod with the wrong image name (like --image=nginx) and find the error message.

28. Get logs for kubectl.

29. Get logs for the scheduler.

30. Restart kubelet.

31. Convert a CRT to a PEM.
    - Convert it back.
    
32. Backup an etcd cluster.

33. List the members of an etcd cluster.
