# CKA Exam Dumps-2

1. Create a node that has a SSD and label it as such.
   
   ```sh
   kubectl label node kubenode01 disktype=ssd
   kubectl label node kubenode01 disktype- # to delete the above label
   
2. Create a pod that is only scheduled on SSD nodes.

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

3. Create 2 pod definitions: the second pod should be scheduled to run anywhere the first pod is running - 2nd pod runs alongside the first pod.

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

   

4. Create a deployment running nginx version 1.12.2 that will run in 2 pods.
   - Scale this to 4 pods.
   - Scale it back to 2 pods.
   - Upgrade this to 1.13.8.
   - Check the status of the upgrade.
   - How do you do this in a way that you can see the history of what happened?
   - Undo the upgrade.
   
   ```sh
   kubectl create deployment my-dep --replicas=2 --image=nginx:1.12.2 --dry-run=client -oyaml > dep.yaml
   kubectl scale --replicas=4 -f dep.yaml 
   kubectl scale --replicas=2 -f dep.yaml 
   kubectl set image deployments/my-dep nginx:1.12.2=nginx:1.13.8  
   kubectl rollout status deployments/my-dep
   kubectl rollout history deployments/my-dep
   kubectl rollout undo deployments/my-dep
   ```
   
5. Create a pod that uses a scratch disk.
   
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
   
   
   
   - Change the pod to mount a disk from the host.
   
     
   
   - Change the pod to mount a persistent volume.
   
   Reference link: https://www.alibabacloud.com/blog/kubernetes-volume-basics-emptydir-and-persistentvolume_594834
   
6. Create a pod that has a liveness check.

7. Create a service that manually requires endpoint creation - and create that too.

8. Create a daemon set.
   - Change the update strategy to do a rolling update but delaying 30 seconds between pod updates.
   
9. Create a static pod.

10. Create a busybox container without a manifest. Then edit the manifest.

11. Create a pod that uses secrets.
    - Pull secrets from environment variables.
    - Pull secrets from a volume.
    - Dump the secrets out via kubectl to show it worked.
    
12. Create a job that runs every 3 minutes and prints out the current time.

13. Create a job that runs 20 times, 5 containers at a time, and prints "Hello parallel world".

14. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx.

15. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%.

16. Create a custom resource definition.
    - Display it in the API with curl.
    
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

34. Create a node that has a SSD and label it as such.

35. Create a pod that is only scheduled on SSD nodes.

36. Create 2 pod definitions: the second pod should be scheduled to run anywhere the first pod is running - 2nd pod runs alongside the first pod.

37. Create a deployment running nginx version 1.12.2 that will run in 2 pods.
    - Scale this to 4 pods.
    - Scale it back to 2 pods.
    - Upgrade this to 1.13.8.
    - Check the status of the upgrade.
    - How do you do this in a way that you can see history of what happened?
    - Undo the upgrade.
    
38. Create a service that uses a scratch disk.

39. Change the service to mount a disk from the host.

40. Change the service to mount a persistent volume.

41. Create a pod that has a liveness check.

42. Create a service that manually requires endpoint creation - and create that too.

43. Create a daemon set.
    - Change the update strategy to do a rolling update but delaying 30 seconds between pod updates.
    
44. Create a static pod.

45. Create a busybox container without a manifest. Then edit the manifest.

46. Create a pod that uses secrets.
    - Pull secrets from environment variables.
    - Pull secrets from a volume.
    - Dump the secrets out via kubectl to show it worked.
    
47. Create a job that runs every 3 minutes and prints out the current time.

48. Create a job that runs 20 times, 5 containers at a time, and prints "Hello parallel world".

49. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx.

50. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%.

51. Create a custom resource definition.
    - Display it in the API with curl.
    
52. Create a networking policy such that only pods with the label access=granted can talk to it.
    - Create an nginx pod and attach this policy to it.
    - Create a busybox pod and attempt to talk to nginx - should be blocked.
    - Attach the label to busybox and try again - should be allowed.
    
53. Create a service that references an externalname.
    - Test that this works from another pod.
    
54. Create a pod that runs all processes as user 1000.

55. Create a namespace.
    - Run a pod in the new namespace.
    - Put memory limits on the namespace.
    - Limit pods to 2 persistent volumes in this namespace.
    
56. Write an ingress rule that redirects calls to /foo to one service and to /bar to another.

57. Write a service that exposes nginx on a nodeport.
    - Change it to use a cluster port.
    - Scale the service.
    - Change it to use an external IP.
    - Change it to use a load balancer.
    
58. Deploy nginx with 3 replicas and then expose a port. Use port forwarding to talk to a specific port.

59. Make an API call using CURL and proper certs.

60. Upgrade a cluster with kubeadm.

61. Get logs for a pod.

62. Deploy a pod with the wrong image name (like --image=nginy) and find the error message.

63. Get logs for kubectl.

64. Get logs for the scheduler.

65. Restart kubelet

66. Convert a CRT to a PEM. Convert it back.

67. Backup an etcd cluster.

68. List the members of an etcd cluster.

69. Find the health of etcd
