# Running a Kubernetes Web Dashboard <code><img src="https://cdn.svgporn.com/logos/kubernetes.svg" height="36"/></code> 

Open two tabs in PowerShell.

First apply the `recommended.yaml` on your cluster.

```powershell
🐳 » kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.1.0/aio/deploy/recommended.yaml
namespace/kubernetes-dashboard unchanged
serviceaccount/kubernetes-dashboard unchanged
service/kubernetes-dashboard unchanged
secret/kubernetes-dashboard-certs unchanged
secret/kubernetes-dashboard-csrf unchanged
secret/kubernetes-dashboard-key-holder unchanged
configmap/kubernetes-dashboard-settings unchanged
role.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard unchanged
deployment.apps/kubernetes-dashboard configured
service/dashboard-metrics-scraper unchanged
deployment.apps/dashboard-metrics-scraper configured
```

```powershell
🐳 » kubectl get ns
NAME                   STATUS   AGE
default                Active   6d2h
kube-node-lease        Active   6d2h
kube-public            Active   6d2h
kube-system            Active   6d2h
kubernetes-dashboard   Active   31m   👈
```

```powershell
🐳 » kubectl -n kubernetes-dashboard get pods -o wide
NAME                                         READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
dashboard-metrics-scraper-79c5968bdc-5wf9j   1/1     Running   0          27m   10.1.0.51   docker-desktop   <none>           <none>
kubernetes-dashboard-7448ffc97b-xj8t9        1/1     Running   0          27m   10.1.0.50   docker-desktop   <none>           <none>
```

```powershell
🐳 » kubectl -n kubernetes-dashboard get svc -o wide
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE   SELECTOR
dashboard-metrics-scraper   ClusterIP   10.105.46.175    <none>        8000/TCP   33m   k8s-app=dashboard-metrics-scraper
kubernetes-dashboard        ClusterIP   10.106.233.212   <none>        443/TCP    33m   k8s-app=kubernetes-dashboard
```

Now, we run the UI for which Token is required.

```powershell
🐳 » kubectl proxy
Starting to serve on 127.0.0.1:8001
```

In the other tab, we need to do the following:

1. We need to create a `service account` in the `default` namespace.

```powershell
🐳 »  kubectl create serviceaccount dashboard -n default
```

2. We then create a `clusterrrolebinding` for the `dashboard` account that was created in `default` namespace.

```powershell
🐳 » kubectl create clusterrolebinding dashboard-admin -n default  --clusterrole=cluster-admin  --serviceaccount=default:dashboard
```

3. We then copy a secret token from the output and enter it in the dashboard. 

   - `Linux`

   ```powershell
   🐳 » kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
   ```

   - `Windows`

   ```powershell
   🐳 » kubectl get secret $(kubectl get serviceaccount dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}"
   ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrOUNWMk52TkVRMVdqQnpTR2RPV0hkcFRGSkVZM042ZUdwbFltcFdSWG93Um1oVVFVTlpUVTF5VWxFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJoYzJoaWIyRnlaQzEwYjJ0bGJpMDJkbVJvYkNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZ5ZG1salpTMWhZMk52ZFc1MExtNWhiV1VpT2lKa1lYTm9ZbTloY21RaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNTFhV1FpT2lKbU5XVmlORGxsTmkxaFkyRTFMVFJqT1RrdE9EazVOR1ZtWVhWc2REcGtZWE5vWW05aGNtUWlmUS5ydHdWTzFaa2lURi14MTBfRzFkUjdlNjJ6clRMNkdMMHJUUG9VRE9RTF9kaWNqRGM1U3JUNGg0bmtwYjN6Z25TWFdXN1g0ejBicmh4dk1tamtmc0dFQkpOdzY2LXBicy1yVHVuanBLQ00zMzQtZnBkazkwa3dBb0xCaHNTMHZuTzdKcDllaDZYeVJ0dTl4MVIyNlhHVEsydmNNVEI0VGZpS0NWZk5FdGVCeTd0TDRKcU9yXzVGeHBMczd1bEQzSElEXzFxbFQtRzZ1S2xwa3hCM1RuMWhGVDBJenZUamtsOURBa0lVeUszc3FERE9VWkt1YkZMUUNqZ0dJMV93TXVtQm9ld09HZWd3Z3FKTHhwU0VVSl94aGw1bldBcUxzRXM1VlRwQVQybUtwalpfdkN1S0U4MjVzbnNOOXlkRDdXMnYxWWlBZ0c4WmdLWUtLbGRESk02eXc=
   ```

   Copy this output into the following command: `[Text.Encoding]::Utf8.GetString([Convert]::FromBase64String('<<<ENCODE_TEXT>>>'))`

   ```powershell
   🐳 » [Text.Encoding]::Utf8.GetString([Convert]::FromBase64String( 'ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrOUNWMk52TkVRMVdqQnpTR2RPV0hkcFRGSkVZM042ZUdwbFltcFdSWG93Um1oVVFVTlpUVTF5VWxFaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbVJoYzJoaWIyRnlaQzEwYjJ0bGJpMDJkbVJvYkNJc0ltdDFZbVZ5Ym1WMFpYTXVhVzh2YzJWeWRtbGpaV0ZqWTI5MWJuUXZjMlZ5ZG1salpTMWhZMk52ZFc1MExtNWhiV1VpT2lKa1lYTm9ZbTloY21RaUxDSnJkV0psY201bGRHVnpMbWx2TDNObGNuWnBZMlZoWTJOdmRXNTBMM05sY25acFkyVXRZV05qYjNWdWRDNTFhV1FpT2lKbU5XVmlORGxsTmkxaFkyRTFMVFJqT1RrdE9EazVOUzFpTTJRelltSXlNekZoWXpZaUxDSnpkV0lpT2lKemVYTjBaVzA2YzJWeWRtbGpaV0ZqWTI5MWJuUTZaR1ZtWVhWc2REcGtZWE5vWW05aGNtUWlmUS5ydHdWTzFaa2lURi14MTBfRzFkUjdlNjJ6clRMNkdMMHJUUG9VRE9RTF9kaWNqRGM1U3JUNGg0bmtwYjN6Z25TWFdXN1g0ejBicmh4dk1tamtmc0dFQkpOdzY2LXBicy1yVHVuanBLQ00zMzQtZnBkazkwa3dBb0xCaHNTMHZuTzdKcDllaDZYeVJ0dTl4MVIyNlhHVEsydmNNVEI0VGZpS0NWZk5FdGVCeTd0TDRKcU9yXzVGeHBMczd1bEQzSElEXzFxbFQtRzZ1S2xwa3hCM1RuMWhGVDBJenZUamtsOURBa0lVeUszc3FERE9VWkt1YkZMUUNqZ0dJMV93TXVtQm9ld09HZWd3Z3FKTHhwU0VVSl94aGw1bldBcUxzRXM1VlRwQVQybUtwalpfdkN1S0U4MjVzbnNOOXlkRDdXMnYxWWlBZ0c4WmdLWUtLbGRESk02eXc='))
   eyJhbGciOiJSUzI1NiIsImtpZCI6Ik9CV2NvNEQ1WjBzSGdOWHdpTFJEY3N6eGplYmpWRXowRmhUQUNZTU1yUlEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRhc2hib2FyZC10b2tlbi02dmRobCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJkYXNoYm9hcmQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJmNWViNDllNi1hY2E1LTRjOTktODk5NS1iM2QzYmIyMzFhYzYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpkYXNoYm9hcmQifQ.rtwVO1ZkiTF-x10_G1dR7e62zrTL6GL0rTPoUDOQL_dicjDc5SrT4h4nkpb3zgnSXWW7X4z0brhxvMmjkfsGEBJNw66-pbs-rTunjpKCM334-fpdk90kwAoLBhsS0vnO7Jp9eh6XyRtu9x1R26XGTK2vcMTB4TfiKCVfNEteBy7tL4JqOr_5FxpLs7ulD3HID_1qlT-G6uKlpkxB3Tn1hFT0IzvTjkl9DAkIUyK3sqDDOUZKubFLQCjgGI1_wMumBoewOGegwgqJLxpSEUJ_xhl5nWAqLsEs5VTpAT2mKpjZ_vCuKE825snsN9ydD7W2v1YiAgG8ZgKYKKldDJM6yw
   ```

This is the token. which we can paste in the Dashboard and *viola*, you a K8s dashboard.