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

## Worker Node Failure

## Networking Failure