sudo systemctl start mysql

sudo /usr/bin/mysql -u root -p

ALTER USER 'root'@'localhost' IDENTIFIED BY 'PASSWORD';

kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -l -n database

kubectl expose deployment restapi -n restapi --port=8080 --dry-run=client -oyaml > service.yaml

kubectl port-forward -n restapi svc/restapi 8080

kubectl exec -it -n database mysql-6df55bffdc-h5j9s bash -- mysql -u root --password=BASE64_DECODED_VALUE_FROM_SECRET

echo ENCODED_VALUE_FROM_SECRET | base64 -d
