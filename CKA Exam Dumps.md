# CKA Exam Dumps 

1. **List your `PersistentVolume` by `Name`, `Size`.**

   ```powershell
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
   
   # creating pv 
   ➜ kubectl create -f pv-volume.yaml 
   persistentvolume/talex-pv-volume created
   persistentvolume/ruby-pv-volume created
   
   # sorting pv by name
   ➜ kubectl get pv --sort-by=.metadata.name
   NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   ruby-pv-volume    78Mi       RWO            Retain           Available           manual                  2m11s
   talex-pv-volume   52Mi       RWO            Retain           Available           manual                  2m11s
   
   # sorting pv by size
   ➜ kubectl get pv --sort-by=.spec.capacity.storage
   NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
   talex-pv-volume   52Mi       RWO            Retain           Available           manual                  15s
   ruby-pv-volume    78Mi       RWO            Retain           Available           manual                  15s
   
   ➜ kubectl get pv --sort-by=.metadata.name,.spec.capacity.storage
   
   ```

**2. All ready nodes, one of them marked as no-schedule, list all the nodes excluding the label no-schedule.**

```powershell
➜ kubectl taint nodes node1 nodeEffect=unput:NoSchedule.
```

**3. kubectl command failing with error - “ The connection to the server localhost:8080 was refused - did you specify the right host or port?”. Fix it.**

- Browse through the `/etcd/kubernetes/manifests/kube-apiserver.yaml`

**4. Create 2 pods -- mypod1 and mypod2 , mypod2 should be scheduled to run anywhere mypod1 is running.  **

1. We can label a node.

   ```sh
   kubectl label node node01 area=selected
   ```

   Put both this label under `nodeSelector` field in the manifest of the node `node01`.

2. We can also taint a particular node and add toleration to the two pods.

3. We can use Node Affinity.

4. We can also use PodAffinity.

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

**5. Questions on Init Containers :**

**1.**   **A container will start only if a file is present. **



**6. Create a deployment running nginx version 1.12.2 that will run in 2 pods  
 a. Scale this to 4 pods.
 b. Scale it back to 2 pods.
 c. Upgrade this to 1.13.8
 d. Check the status of the upgrade.
 e. How do you do this in a way that you can see history of what happened?
 f. Undo the upgrade.**

 

**7. Create a pod that has a liveness check.  **

 

**8. Create a pod that has a readiness check.** 

 

**9. Create a busybox container without a manifest. Then edit the manifest.  **

 

**10. Create a job that runs every 3 minutes and prints out the current time.  **

 

**11. Create a job that runs 20 times, 5 containers at a time, and prints "Hello parallel world"  **

 

**12. Get the list of pod by doing a CURL to the kube-apiserver.  **

 

**13. Deploy a pod with the wrong image name (like --image=nginy) and find the error message.  **

 

**14. Get logs for a pod which has multiple containers running.  **

 

**15. Deploy nginx with 3 replicas and then expose a port  
 a. Use port forwarding to talk to a specific port**

 

**16. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx.  **

 

**17. Get the status of all the master components  **

 

**18.Create a pod that runs on a given node. **

 

**19. Create a pod that uses secrets  
 a. Pull secrets from environment variables
 b. Pull secrets from a volume
 c. Dump the secrets out via kubectl to show it worked**

 

**20. Create a static pod and then delete the pod.  **

 

**21. Create a pod that do not get IP from the range of allocated CIDR block. Ensure that this is not a static pod.  **

 

**22. Create a service that uses a scratch disk.  
 a. Change the service to mount a disk from the host. [Local-PV]
 b. Change the service to mount a persistent volume. [hostPath PV]**

 

**24. Create a service that manually requires endpoint creation - and create that too.  **

 

**25. Create a daemon set
 a. Change the update strategy to do a rolling update but delaying 30 seconds  **

 

**26. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%.  **

 

**27. Create a custom resource definition  
 a. Display it in the API with curl**

 

**28. Create a service that references an externalname.  
 a. Test that this works from another pod**

 

**29. Create a pod that runs all processes as user 1000.  **

 

**30. Write an ingress rule that redirects calls to /foo to one service and to /bar to another. **

 

**31. Write a service that exposes nginx on a nodeport  
 a. Change it to use a cluster port
 b. Scale the service
 c. Change it to use an external IP
 d. Change it to use a load balancer**

 

**32. Deploy nginx with 3 replicas and then expose a port  
 a. Use port forwarding to talk to a specific port**

 

**33. Get logs for Kubernetes master components  **

 

**34. Get logs for Kubelet.  **

 

**35. Backup an etcd cluster  **

 

**36. List the members of an etcd cluster **

 

**37. Find the health of etcd  **

 

**38. Create a namespace [Important]  
 a. Run a pod in the new namespace
 b. Put memory limits on the namespace
 c. Limit pods to 2 persistent volumes in this namespace**

 

**39. Create a networking policy such that only pods with the label access=granted can talk to it.  
 a. Create an nginx pod and attach this policy to it. 
 b. Create a busybox pod and attempt to talk to nginx - should be blocked
 c. Attach the label to busybox and try again - should be allowed**

 

**40. Create a multi containers of nginx, redis and consul. **

 

**41. Troubleshooting not ready state node. **

 

**42. Add missing worker node -- TLS bootstrapping. **

 

**43. Set up a Kubernetes cluster from scratch by using Kubeadm  **

 

**44. Create Redis pod with non-pvolume.  **

 

**45. Creating PVolume with host path.  **

 

**46. Create pods,service in particular namespace, list all services in particular namespace. **

 

**47. Create nginx deployment nginx-random expose it**

**then create another pod busybox and do the following:  **

**(a) Dnlookup service**

**(b) Dnslookup pod.**

 

**48. Expose a service to Nodeport.  **

 

**49. Create a pod that by passes kube-scheduler. Ensure that this is not a static pod.  **

######  