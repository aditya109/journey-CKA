# Application Lifecycle Management

## Rolling Updates and Rollbacks in Deployments

### Rollout and Versioning

![](https://raw.githubusercontent.com/aditya109/learning-k8s/main/Application Lifecycle Management/assets/rollouts.svg)

Suppose we have multiple sets of applications running in our cluster, and let's say we are on *version* `Revision 1` of the application. We now have to upgrade all of those running applications to `Revision 2`. The process of upgrading all our running applications to required version is called **rollout**.

```powershell
kubectl rollout status deployment/myapp-deployment
```

The above command shows the status of the pushed rollout.

```powershell
kubectl rollout history deployment/myapp-deployment
```

The above command shows the revisions and history of our deployment.

### Deployment Strategy





## Configure Applications

## Scale Applications

## Self-Healing Application