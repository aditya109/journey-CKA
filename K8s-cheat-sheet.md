1. To get the yaml manifest for a running Kubernetes object, 

   ```powershell
   kubectl get pod webapp_color -o yaml > pod1.yaml
   ```

2. To create an alias in Linux: `alias k8s="kubectl"`

3. Changing namespace:

   ```powershell
   kubectl config set-context --current --namespace=<insert-namespace-name-here>
   # Validate it
   kubectl config view --minify | grep namespace:
   ```

4. Creating a config map:

   ```powershell
   kubectl create configmap \
   	<config-name> --from-literal=<key>=<value>
   ```

   

   