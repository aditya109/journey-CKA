# CKA Exam Dumps-2

1. Create a node that has a SSD and label it as such.
2. Create a pod that is only scheduled on SSD nodes.
3. Create 2 pod definitions: the second pod should be scheduled to run anywhere the first pod is running - 2nd pod runs alongside the first pod.
4. Create a deployment running nginx version 1.12.2 that will run in 2 pods.
   - Scale this to 4 pods.
   - Scale it back to 2 pods.
   - Upgrade this to 1.13.8.
   - Check the status of the upgrade.
   - How do you do this in a way that you can see the history of what happened?
   - Undo the upgrade.
5. Create a service that uses a scratch disk.
   - Change the service to mount a disk from the host.
   - Change the service to mount a persistent volume.
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
