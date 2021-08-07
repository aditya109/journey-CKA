# CKA Exam Dumps 

1. **List your `PersistentVolume` by `Name`, `Size`.**

   

2. ajhnasuncujunyhhhjkinjuiihjkjkkkkigghjlasudksd

**CKA Questions [Confidential]**

 

**1. List your pv sort by name.[DONE]**

  **List your pv sort by size.[DONE]**

 

**2. All ready nodes, one of them marked as no-schedule, list all the nodes excluding the label no-schedule.[DONE]**

 

**3. Kubectl command failing with error - “ The connection to the server localhost:8080 was refused - did you specify the right host or port?”. Fix it.**

 

**4. Create 2 pods -- mypod1 and mypod2 , mypod2 should be scheduled to run anywhere mypod1 is running. [DONE]**

 

**5. Questions on Init Containers :**

**1.**   **A container will start only if a file is present. [DONE]**

 

**6. Create a deployment running nginx version 1.12.2 that will run in 2 pods [DONE]
 a. Scale this to 4 pods.
 b. Scale it back to 2 pods.
 c. Upgrade this to 1.13.8
 d. Check the status of the upgrade.
 e. How do you do this in a way that you can see history of what happened?
 f. Undo the upgrade.**

 

**7. Create a pod that has a liveness check. [DONE]**

 

**8. Create a pod that has a readiness check.** 

 

**9. Create a busybox container without a manifest. Then edit the manifest. [DONE]**

 

**10. Create a job that runs every 3 minutes and prints out the current time. [DONE]**

 

**11. Create a job that runs 20 times, 5 containers at a time, and prints "Hello parallel world" [DONE]**

 

**12. Get the list of pod by doing a CURL to the kube-apiserver. [Done]**

 

**13. Deploy a pod with the wrong image name (like --image=nginy) and find the error message. [DONE]**

 

**14. Get logs for a pod which has multiple containers running. [DONE]**

 

**15. Deploy nginx with 3 replicas and then expose a port [DONE]
 a. Use port forwarding to talk to a specific port**

 

**16. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx. [DONE]**

 

**17. Get the status of all the master components [DONE]**

 

**18.Create a pod that runs on a given node.[DONE]**

 

**19. Create a pod that uses secrets [DONE]
 a. Pull secrets from environment variables
 b. Pull secrets from a volume
 c. Dump the secrets out via kubectl to show it worked**

 

**20. Create a static pod and then delete the pod. [DONE]**

 

**21. Create a pod that do not get IP from the range of allocated CIDR block. Ensure that this is not a static pod. [DONE]**

 

**22. Create a service that uses a scratch disk. [Done]
 a. Change the service to mount a disk from the host. [Local-PV]
 b. Change the service to mount a persistent volume. [hostPath PV]**

 

**24. Create a service that manually requires endpoint creation - and create that too. [DONE]**

 

**25. Create a daemon set
 a. Change the update strategy to do a rolling update but delaying 30 seconds [DONE]**

 

**26. Create a horizontal autoscaling group that starts with 2 pods and scales when CPU usage is over 50%. [Done]**

 

**27. Create a custom resource definition [DONE]
 a. Display it in the API with curl**

 

**28. Create a service that references an externalname. [DONE]
 a. Test that this works from another pod**

 

**29. Create a pod that runs all processes as user 1000. [DONE]**

 

**30. Write an ingress rule that redirects calls to /foo to one service and to /bar to another.[Done]**

 

**31. Write a service that exposes nginx on a nodeport [DONE]
 a. Change it to use a cluster port
 b. Scale the service
 c. Change it to use an external IP
 d. Change it to use a load balancer**

 

**32. Deploy nginx with 3 replicas and then expose a port [DONE]
 a. Use port forwarding to talk to a specific port**

 

**33. Get logs for Kubernetes master components [DONE]**

 

**34. Get logs for Kubelet. [DONE]**

 

**35. Backup an etcd cluster [DONE]**

 

**36. List the members of an etcd cluster[DONE]**

 

**37. Find the health of etcd [DONE]**

 

**38. Create a namespace [Important] [Done]
 a. Run a pod in the new namespace
 b. Put memory limits on the namespace
 c. Limit pods to 2 persistent volumes in this namespace**

 

**39. Create a networking policy such that only pods with the label access=granted can talk to it. [DONE]
 a. Create an nginx pod and attach this policy to it. 
 b. Create a busybox pod and attempt to talk to nginx - should be blocked
 c. Attach the label to busybox and try again - should be allowed**

 

**40. Create a multi containers of nginx, redis and consul.[DONE]**

 

**41. Troubleshooting not ready state node.[DONE]**

 

**42. Add missing worker node -- TLS bootstrapping.[DONE]**

 

**43. Set up a Kubernetes cluster from scratch by using Kubeadm [DONE]**

 

**44. Create Redis pod with non-pvolume. [DONE]**

 

**45. Creating PVolume with host path. [Done]**

 

**46. Create pods,service in particular namespace, list all services in particular namespace.[DONE]**

 

**47. Create nginx deployment nginx-random expose it**

**then create another pod busybox and do the following: [DONE]**

**(a) Dnlookup service**

**(b) Dnslookup pod.**

 

**48. Expose a service to Nodeport. [Done]**

 

**49. Create a pod that by passes kube-scheduler. Ensure that this is not a static pod. [DONE]**

######  