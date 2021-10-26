# Troubleshooting

## Contents

## Application Failure

- Check Accessibility

  ![](https://github.com/aditya109/learning-k8s/blob/main/assets/app%20failure.svg?raw=true)

  

  Let's assume the user reported some issue with accessing the application.

  - First we check if we can access `web server` by doing a `cURL`.
  
    ```bash
    curl http://web-service-ip:node-port
    ```
  
  - Then check the `web service`.
  
    ```bash
    kubectl describe service web-service
    ```
  
    > A good idea would be to compare selectors of `web service` and `web pod` here.
  
  - Then check the `web pod`.
  
    > Check its status and also check its logs from `kubectl describe pod web`.
    > Also check its logs `kubectl logs web` or running logs using `kubectl logs web -f` or check the previous pod logs using `kubectl logs web -f --previous`.
  
  - Similarly check `db service` and `db pod`.

## Control Plan Failure

1. Check Node status.

   ```sh
   kubectl get nodes
   ```

2. Check status of pods.

   ```sh
   kubectl get pods
   ```

3. Check controlplane pods.

   ```sh
   kubectl get pods -n kube-system
   ```

   If deployed as services, check the status of the services.

   ```sh
   service kube-apiserver status
   service kube-controller-manager status
   service kube-scheduler status
   ```

   On the worker nodes,

   ```sh
   service kubelet status
   service kube-proxy status
   ```

4. Check logs of the controlplane components.

   ```sh
   kubectl logs kube-apiserver-master -n kube-system
   ```

   We can also use `journalctl` utility.

   ```sh
   sudo journalctl -u kube-apiserver
   ```

   

## Worker Node Failure

## Networking Failure