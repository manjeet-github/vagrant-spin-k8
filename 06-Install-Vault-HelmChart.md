## - Install Vault Helm chart. Install Vault Server and Vault Injector in k8

```
⇒  helm install vault \
    --set "server.dev.enabled=true" \
    https://github.com/hashicorp/vault-helm/archive/v0.5.0.tar.gz
NAME: vault
LAST DEPLOYED: Fri Jun 19 10:52:51 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/


Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get vault
  

- Check if the helm install started the Vault Server POD & Vault-Injector POD. 
- SIGN Of Success
⇒  kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          31s
vault-agent-injector-7cf6975f99-97grp   1/1     Running   0          31s
```


## -- Create Secrets (Vault is running on K8 - use port-forwarding)
```
- You can connect to the pod and run the vault commands to create secrets.
- Other option is to do port-forwarding which will give you both CLI & UI access

⇒  kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          11m
vault-agent-injector-7cf6975f99-97grp   1/1     Running   0          11m


⇒  kubectl port-forward vault-0 28200:8200
Forwarding from 127.0.0.1:28200 -> 8200
Forwarding from [::1]:28200 -> 8200

⇒  export VAULT_TOKEN=root
⇒  export VAULT_ADDR=http://127.0.0.1:28200
⇒  vault secrets list
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_c55f0b7e    per-token private secret storage
identity/     identity     identity_eb719c5f     identity store
secret/       kv           kv_c4fd7a15           key/value secret storage
sys/          system       system_7d6fea42       system endpoints used for control, policy and debugging

Access the UI - http://127.0.0.1:28200/ui

⇒  vault secrets enable -path=internal kv-v2
Success! Enabled the kv-v2 secrets engine at: internal/

⇒  vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"
Key              Value
---              -----
created_time     2020-06-19T15:12:40.676315611Z
deletion_time    n/a
destroyed        false
version          1

⇒  vault kv get internal/database/config
====== Metadata ======
Key              Value
---              -----
created_time     2020-06-19T15:12:40.676315611Z
deletion_time    n/a
destroyed        false
version          1

====== Data ======
Key         Value
---         -----
password    db-secret-password
username    db-readonly-username

⇒  vault read internal/data/database/config
Key         Value
---         -----
data        map[password:db-secret-password username:db-readonly-username]
metadata    map[created_time:2020-06-19T15:12:40.676315611Z deletion_time: destroyed:false version:1]

```

## - Configure K8-Auth in our vault-0
```
⇒  export KUBECONFIG=/Users/manjeet/.kube/config:/Users/manjeet/.kube/custom-contexts/vault-k8-cluster-config
⇒  export K8_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}')
⇒  export SA_JWT_TOKEN=$(kubectl get secret \
  $(kubectl get serviceaccount vault-reviewer -o json | jq -r '.secrets[0].name') \
  -o json | jq -r '.data .token' | base64 -D -)

⇒  kubectl get serviceaccount vault-reviewer -o json | jq -r '.secrets[0].name'
vault-reviewer-token-5wdkq

⇒  kubectl get secret vault-reviewer-token-5wdkq -o jsonpath="{.data['ca\.crt']}" | base64 --decode > k8-ca.crt

⇒  env | grep VAULT
VAULT_TOKEN=root
VAULT_ADDR=http://127.0.0.1:28200

⇒  vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/

⇒  vault write auth/kubernetes/config \
    token_reviewer_jwt="${SA_JWT_TOKEN}"  \
    kubernetes_host="${K8_HOST}" \
    kubernetes_ca_cert=@k8-ca.crt
Success! Data written to: auth/kubernetes/config

⇒  vault read auth/kubernetes/config
Key                   Value
---                   -----
issuer                n/a
kubernetes_ca_cert    -----BEGIN CERTIFICATE-----
MIICyDCCAbCgAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
cm5ldGVzMB4XDTIwMDYxODIwMTgzMFoXDTMwMDYxNjIwMTgzMFowFTETMBEGA1UE
AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANsb
YnGduxztc6OFqTIkiG4AOHbMl8JJS72fWTNDvsLKhDXTRxk1+781niHHRfvW7BKr
Eq+yulAUYBcO4WL6CtQ2ispg1VQUorniBIxkc3g72KU68pGshejktLgqKksnt3Nn
CAs1ikiphdQDnWM2rycgPmX1KaE5JgUp/amYePe0+/jttLkh7+J1NoDzTzzacMNC
Ystah5iZd6x/FZ+C7+EyBAHi/NPBlrfLCuB7yrHKuVdK5o30hASGibHoVSR1idLp
mlpxZxudSnJT/UC3N7pMvtaDRuSWVDQsJ+MMiOg/HDZuwMWUAA/bLT0xK5vFic1Q
qlFLPgZSHbmXsgwFL3UCAwEAAaMjMCEwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
/wQFMAMBAf8wDQYJKoZIhvcNAQELBQADggEBAGXB8UGN9AyKCg/NepUOt6eylJ5A
hn30BZ+JIMCaBZ7KfrRx15ceVrd/IRZzfTx7KkjzSwBSW9Y1itxYZn9aOL8quw9J
EAGdbua6szwagt+2N2RHKDqXc/YVyiUmLn4qjLofcFv+jRVeHo6iRbKwVcufVer4
6k3KgloX+weR2mGKd9YwmHCsIt4j6bPjNFgpnUBAe0sKMbaPN5cpy1IRA7A+nKYt
nFxPRcVVR4wpGp8XmRGxT7h3civwvHXDxqy7vM4w7YN4bNMBwXyI6PVAqQTYa/Kk
Zw+zsMxwuUewlsHjsrX+Vxthm7jHtQOZgpaiHvvmnSXAtnF1elChMllZDLQ=
-----END CERTIFICATE-----
kubernetes_host       https://172.42.42.100:6443
pem_keys              []

```


## - Deploy our APP
```
-- Four step Process
-- Step1 - Setup SA for the app in k8
-- Step2 - Setup a policy to access secrets. this will be used for setting k8-auth role
-- Step3 - Setup a k8-auth role
-- Step4 - Deploy your app by referencing annotations.

-- Create Vault Policy
⇒  cat internal-app-policy.hcl
path "internal/data/database/config" {
  capabilities = ["read"]
}

⇒  vault policy write internal-app yaml-files/internal-app-policy.hcl
Success! Uploaded policy: internal-app


-- Create k8-AUTH Role
⇒  vault write auth/kubernetes/role/internal-app \
    bound_service_account_names=internal-app \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=24h
Success! Data written to: auth/kubernetes/role/internal-app
```

-- Create internal-app SA and Deploy the App
```
⇒  cat 05-deploy-orgchart-app.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orgchart
  labels:
    app: orgchart
spec:
  selector:
    matchLabels:
      app: orgchart
  replicas: 1
  template:
    metadata:
      annotations:
      labels:
        app: orgchart
    spec:
      serviceAccountName: internal-app
      containers:
        - name: orgchart
          image: jweissig/app:0.0.1

⇒  kubectl create -f yaml-files/05-deploy-orgchart-app.yaml
serviceaccount/internal-app created
deployment.apps/orgchart created

NOTE - THE APP IS DEPLOYED BY NO ANNOTATIONS ARE DEFINED TO USE VAULT INJECTOR
CHECK THE NUMBER OF CONTAINERS RUNNING IN ORGCHART APP
⇒  kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-7f6b86f74f-7zvhn               1/1     Running   0          2m15s
vault-0                                 1/1     Running   0          54m
vault-agent-injector-7cf6975f99-97grp   1/1     Running   0          54m

```


## - Deploy the APP patch with Annotations
```
⇒  kubectl patch deployment orgchart --patch "$(cat yaml-files/06-deploy-orgchart-app-patch.yaml)"
deployment.apps/orgchart patched

-- NOTE - Check the number of containers starting in the new deployment
⇒  kubectl get pods
NAME                                    READY   STATUS            RESTARTS   AGE
orgchart-7654cd56f9-5rb75               0/2     PodInitializing   0          5s
orgchart-7f6b86f74f-7zvhn               1/1     Running           0          4m23s
vault-0                                 1/1     Running           0          57m
vault-agent-injector-7cf6975f99-97grp   1/1     Running           0          57m

⇒  kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-7654cd56f9-5rb75               2/2     Running   0          52s
vault-0                                 1/1     Running   0          57m
vault-agent-injector-7cf6975f99-97grp   1/1     Running   0          57m
```


## - Let check if the new deployed app got the secrets from Vault
```
⇒  kubectl logs \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container vault-agent
==> Vault server started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.4.0

2020-06-19T15:49:54.844Z [INFO]  sink.file: creating file sink
2020-06-19T15:49:54.845Z [INFO]  sink.file: file sink configured: path=/home/vault/.vault-token mode=-rw-r-----
2020-06-19T15:49:54.845Z [INFO]  template.server: starting template server
2020/06/19 15:49:54.845431 [INFO] (runner) creating new runner (dry: false, once: false)
2020/06/19 15:49:54.845786 [INFO] (runner) creating watcher
2020-06-19T15:49:54.845Z [INFO]  auth.handler: starting auth handler
2020-06-19T15:49:54.845Z [INFO]  auth.handler: authenticating
2020-06-19T15:49:54.846Z [INFO]  sink.server: starting sink server
2020-06-19T15:49:54.862Z [INFO]  auth.handler: authentication successful, sending token to sinks
2020-06-19T15:49:54.862Z [INFO]  auth.handler: starting renewal process
2020-06-19T15:49:54.863Z [INFO]  sink.file: token written: path=/home/vault/.vault-token
2020-06-19T15:49:54.863Z [INFO]  template.server: template server received new token
2020/06/19 15:49:54.863410 [INFO] (runner) stopping
2020/06/19 15:49:54.863433 [INFO] (runner) creating new runner (dry: false, once: false)
2020/06/19 15:49:54.863506 [INFO] (runner) creating watcher
2020/06/19 15:49:54.863540 [INFO] (runner) starting
2020-06-19T15:49:54.872Z [INFO]  auth.handler: renewed auth token




⇒  kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    --container orgchart -- cat /vault/secrets/database-config.txt
data: map[password:db-secret-password username:db-readonly-username]
metadata: map[created_time:2020-06-19T15:12:40.676315611Z deletion_time: destroyed:false version:1]


```



## - Deploy APP again. Use Annotation to use templates definition
```
⇒  cat 07-deploy-orgchart-app-patch-template.yaml
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/role: "internal-app"
        vault.hashicorp.com/agent-inject-secret-database-config.txt: "internal/data/database/config"
        vault.hashicorp.com/agent-inject-template-database-config.txt: |
          {{- with secret "internal/data/database/config" -}}
          postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard
          {{- end -}}


⇒  kubectl patch deployment orgchart --patch "$(cat 07-deploy-orgchart-app-patch-template.yaml)"
deployment.apps/orgchart patched

⇒  kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
orgchart-6cc49fd78f-zs6lt               2/2     Running       0          5s
orgchart-7654cd56f9-5rb75               0/2     Terminating   0          6m28s
vault-0                                 1/1     Running       0          63m
vault-agent-injector-7cf6975f99-97grp   1/1     Running       0          63m

⇒  kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
orgchart-6cc49fd78f-zs6lt               2/2     Running   0          37s
vault-0                                 1/1     Running   0          63m
vault-agent-injector-7cf6975f99-97grp   1/1     Running   0          63m

⇒  kubectl exec \
    $(kubectl get pod -l app=orgchart -o jsonpath="{.items[0].metadata.name}") \
    -c orgchart -- cat /vault/secrets/database-config.txt
postgresql://db-readonly-username:db-secret-password@postgres:5432/wizard%

```


## Uninstall the apps and helm chart
```
⇒  kubectl delete -f 05-deploy-orgchart-app.yaml
serviceaccount "internal-app" deleted
deployment.apps "orgchart" deleted

⇒  kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          69m
vault-agent-injector-7cf6975f99-97grp   1/1     Running   0          69m

⇒  helm list
NAME 	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART      	APP VERSION
vault	default  	1       	2020-06-19 10:52:51.599095 -0400 EDT	deployed	vault-0.5.0

⇒  helm uninstall vault
release "vault" uninstalled

⇒  kubectl get pods
NAME                                    READY   STATUS        RESTARTS   AGE
vault-0                                 1/1     Terminating   0          70m
vault-agent-injector-7cf6975f99-97grp   0/1     Terminating   0          70m

```

