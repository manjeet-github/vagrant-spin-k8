# Deploy app with hardcoded Vault Address

## - Create a 01-deploy-app-hardcoded-external-vault.yaml file:
```
cat <<EOF> 01-deploy-app-hardcoded-external-vault.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-hardcoded-external-vault
  labels:
    app: app1-hardcoded-external-vault
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1-hardcoded-external-vault
  template:
    metadata:
      labels:
        app: app1-hardcoded-external-vault
    spec:
      serviceAccountName: vault-auth
      containers:
        - name: app1
          image: "kawsark/vault-example-init:0.0.7"
          imagePullPolicy: Always
          env:
            - name: VAULT_ADDR
              value: "${VAULT_ADDR}"
            - name: VAULT_ROLE
              value: "app1-role"
            - name: SECRET_KEY
              value: "secret/app1"
            - name: VAULT_LOGIN_PATH
              value: "auth/kubernetes/login"
EOF

# Note - Make sure the VAULT_ADDR is reachable from the pods. 
# I am using 172.42.42.200 to connect to vault
# Display contents of deployment.yaml and ensure that the values of 
# these 4 environment variables are correct: VAULT_ADDR, VAULT_ROLE,
# SECRET_KEY and VAULT_LOGIN_PATH. These values will be read by the
# application.

cat 01-deploy-app-hardcoded-external-vault.yaml
```

## - Deploy APP and check that the application pod is running
```
⇒  kubectl create -f yaml-files/01-deploy-app-hardcoded-external-vault.yaml
deployment.apps/app1-hardcoded-external-vault created

⇒  kubectl get pods
NAME                                             READY   STATUS    RESTARTS   AGE
app1-hardcoded-external-vault-6dd57c84b8-446zn   1/1     Running   0          6s

⇒  kubectl logs app1-hardcoded-external-vault-6dd57c84b8-446zn
2020/06/19 04:32:44 ==> WARNING: Don't ever write secrets to logs.
2020/06/19 04:32:44 ==>          This is for demonstration only.
2020/06/19 04:32:44 s.BanxvTM9p9lc00pdQxmTBTNr
2020/06/19 04:32:44 secret secret/app1 -> &{f0f15680-dd86-d029-2478-10763d2314bb  2764800 false map[password:supasecr3t username:app1] [] <nil> <nil>}
2020/06/19 04:32:44 Starting renewal loop
2020/06/19 04:32:44 Successfully renewed: &api.RenewOutput{RenewedAt:time.Time{wall:0x68eb5f6, ext:63728137964, loc:(*time.Location)(nil)}, Secret:(*api.Secret)(0xc000049620)}


The application reads the JWT for app1 service account from the path “/var/run/secrets/kubernetes.io/serviceaccount/token” and uses it to send a login request to Vault. You can see this in the application’s main.go file.


You can also test the k8 auth by connecting to the pod
⇒  kubectl exec -it app1-hardcoded-external-vault-6dd57c84b8-7n68c -- sh
/ #
/ # export VAULT_ROLE=app1-role
/ # export jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
/ # export VAULT_ADDR=http://172.42.42.200:8200
/ # export VAULT_LOGIN="auth/kuberneetes/login"

$ curl --data "{\"role\":\"${VAULT_ROLE}\",\"jwt\":\"${jwt}\"}"  "${VAULT_ADDR}/v1/${VAULT_LOGIN_PATH}" | jq -r
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1611  100   670  100   941  74444   102k --:--:-- --:--:-- --:--:--  174k
{
  "request_id": "17cf4861-dc69-0257-ecc2-bbdac41295e4",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "s.nZuIchRyR6Kqb6MgIho83Vcm",
    "accessor": "S4QK8UEJ51pfa6Vzz6N09B7s",
    "policies": [
      "app1-policy",
      "default"
    ],
    "token_policies": [
      "app1-policy",
      "default"
    ],
    "metadata": {
      "role": "app1-role",
      "service_account_name": "vault-auth",
      "service_account_namespace": "default",
      "service_account_secret_name": "vault-auth-token-srqnz",
      "service_account_uid": "3757f1f3-b119-4878-b7b3-31f1e5c8b1df"
    },
    "lease_duration": 3600,
    "renewable": true,
    "entity_id": "48aecdbe-d43e-7811-718d-97dd1532fec4",
    "token_type": "service",
    "orphan": true
  }
}
/

## - Delete the deployment
```
⇒  kubectl delete -f yaml-files/01-deploy-app-hardcoded-external-vault.yaml
deployment.apps "app1-hardcoded-external-vault" deleted
```
