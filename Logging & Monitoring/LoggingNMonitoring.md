# Logging and Monitoring

**How do monitor resource consumption in Kubernetes ? What would you like to know in a cluster ??**

Maybe the following node-level metrics:

1. Total number of nodes in the cluster.
2. Number of healthy nodes in the cluster.
3. CPU, Memory, Disk utilization.

Maybe the similar pod-related metrics as well.

## Monitoring Solution

As of now, `Kubernetes` does not come with a  fully functional logging and monitoring solution.

But, we do have a number of open-source solutions to do the same:

1. `Metrics Server`(`Heapster` now deprecated)
2. `Prometheus`
3. `Elastic Stack`
4. `Datadog`
5. `dynatrace`

`Metrics Server` is an in-memory logging service, so historical data cannot be viewed.

### How to works?

`kubelet` contains `cAdvisor` which is responsible for exposing the logs of pods.

`kubectl top node` to view node-metrics.

`kubectl top pod` to view pod-metrics.

### How to see live logs from Kubernetes Resources ?

```yaml
# event-simulator.yaml
apiVersion: v1
kind: Pod
metadata:
  name: event-simulator-pod
spec:
  containers:
  - name: event-simulator
    image: kodekloud/event-simulator
```

```powershell
🐳 » kubectl create -f event-simulator.yml
pod/event-simulator-pod created
🐳 » kubectl logs -f event-simulator-pod
[2021-01-25 15:28:30,805] INFO in event-simulator: USER3 logged out
[2021-01-25 15:28:31,806] INFO in event-simulator: USER4 logged out
[2021-01-25 15:28:32,807] INFO in event-simulator: USER4 is viewing page1
[2021-01-25 15:28:33,809] INFO in event-simulator: USER3 is viewing page1
[2021-01-25 15:28:34,810] INFO in event-simulator: USER1 is viewing page3
[2021-01-25 15:28:35,812] WARNING in event-simulator: USER5 Failed to Login as the account is locked due to MANY FAILED ATTEMPTS.
[2021-01-25 15:28:35,812] INFO in event-simulator: USER1 logged out
[2021-01-25 15:28:36,813] INFO in event-simulator: USER2 logged in
[2021-01-25 15:28:37,815] INFO in event-simulator: USER1 is viewing page2
[2021-01-25 15:28:38,816] WARNING in event-simulator: USER7 Order failed as the item is OUT OF STOCK.
[2021-01-25 15:28:38,816] INFO in event-simulator: USER4 is viewing page1
[2021-01-25 15:28:39,818] INFO in event-simulator: USER1 is viewing page3
[2021-01-25 15:28:40,819] WARNING in event-simulator: USER5 Failed to Login as the account is locked due to MANY FAILED ATTEMPTS.
```

These logs are specific to the container running inside the pod.
But let's say we change `event-simulator.yaml` run additional pods.

```yaml
# event-simulator.yaml
apiVersion: v1
kind: Pod
metadata:
  name: event-simulator-pod
spec:
  containers:
  - name: event-simulator
    image: kodekloud/event-simulator
  - name: image-processor
    image: some-image-processor
```

But, what if we run this resource `event-simulator.yaml` again, whose logs would be displayed by `kubectl logs -f event-simulator-pod` ?

Here, we need to specify the corresponding container-name and the relevant container logs will be displayed.

```bash
kubectl logs -f event-simulator-pod event-simulator
```





