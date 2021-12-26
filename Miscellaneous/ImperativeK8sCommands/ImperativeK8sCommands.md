# Imperative Commands

[https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-ingress-em-](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-ingress-em-)

1. Exec into pod container:

   ```powershell
   kubectl -n <namespace1> exec -it <podName>
   ```

2. Creating a Pod via `kubectl`

   ```powershell
   kubectl run <podname> --image=<ImageName> --restart=Never --dry-run=client -o yaml > podname.yaml
   ```

3. To get the yaml manifest for a running Kubernetes object:

   ```powershell
   kubectl get pod webapp_color -o yaml > pod1.yaml
   ```

4. To create an alias in Linux: `alias k8s="kubectl"`

5. Changing namespace:

   ```powershell
   kubectl config set-context --current --namespace=<insert-namespace-name-here>
   # Validate it
   kubectl config view --minify | grep namespace:
   ```

6. Creating a config map:

   ```powershell
   kubectl create configmap \
   	<config-name> --from-literal=<key>=<value>
   ```

7. Drain a node by ignoring `daemonsets`:

   ```bash
   kubectl drain node01 --ignore-daemonsets --force --delete-local-data
   ```

8. Get pods in all namespaces:

   ```bash
   kubectl get pods --all-
   ```

9. Get all resources in a namespace.

   ```bash
   kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -l <label>=<value> -n <namespace>
   ```

10. Inspect the kubelet service and identify the network plugin configured for Kubernetes.

    ```sh
    ps -aux | grep kubelet | grep --color network-plugin=
    ```

11. To check CNI solution applied on the system.

    ```shell
    ll /opt/cni/bin
    ```

12. What is the CNI plugin configured to be used on this kubernetes cluster?

    ```sh
    ls /etc/cni/net.d/
    ```
