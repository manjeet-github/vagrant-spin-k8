## - Deploy k8 egress services endpoint to point external Vault.

```
⇒  cat 02-deploy-vault-external-service.yaml
---
apiVersion: v1
kind: Service
metadata:
  name: external-vault
  namespace: default
spec:
  ports:
  - protocol: TCP
    port: 8200
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-vault
subsets:
  - addresses:
      - ip: 172.42.42.200
    ports:
      - port: 8200


⇒  kubectl create -f yaml-files/02-deploy-vault-external-service.yaml
service/external-vault created
endpoints/external-vault created

⇒  kubectl get services
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
external-vault   ClusterIP   10.97.242.215   <none>        8200/TCP   91s
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP    65m

```



## - Deploy an app to use external Vault to get secrets.
```
⇒  cat 03-deploy-devweb-app.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devwebapp
  labels:
    app: devwebapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devwebapp
  template:
    metadata:
      labels:
        app: devwebapp
    spec:
      serviceAccountName: internal-app
      containers:
      - name: app
        image: burtlo/devwebapp-ruby:k8s
        imagePullPolicy: Always
        env:
        - name: VAULT_ADDR
          value: "http://172.42.42.200:8200"


⇒  kubectl create -f yaml-files/03-deploy-devweb-app.yaml
serviceaccount/internal-app created
deployment.apps/devwebapp created


⇒  kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
devwebapp-56c46857c4-wpxj6   1/1     Running   0          109s

```


## - Test and validate if external Vault URI is accesible from deployed POD
```
⇒  kubectl exec devwebapp-56c46857c4-wpxj6 -- curl -s http://external-vault:8200/v1/sys/seal-status | jq
{
  "type": "shamir",
  "initialized": true,
  "sealed": false,
  "t": 1,
  "n": 1,
  "progress": 0,
  "nonce": "",
  "version": "1.4.0+ent.hsm",
  "migration": false,
  "cluster_name": "vault-cluster-ddd38052",
  "cluster_id": "7e720e59-280e-55df-c4d1-134f940a32d8",
  "recovery_seal": true,
  "storage_type": "file"
}

```


## - Install the Vault-Injector HELM chart
- Install only Vault-Injector. Don't install Vault Server helm chart
```
⇒  helm version
version.BuildInfo{Version:"v3.2.0", GitCommit:"e11b7ce3b12db2941e90399e874513fbd24bcb71", GitTreeState:"clean", GoVersion:"go1.13.10"}


⇒  helm list
NAME	NAMESPACE	REVISION	UPDATED	STATUS	CHART	APP VERSION


⇒  helm install vault \
    --set "injector.externalVaultAddr=http://external-vault:8200" \
    https://github.com/hashicorp/vault-helm/archive/v0.5.0.tar.gz
NAME: vault
LAST DEPLOYED: Fri Jun 19 02:42:46 2020
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


⇒  helm list
NAME 	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART      	APP VERSION
vault	default  	1       	2020-06-19 13:57:25.942879 -0400 EDT	deployed	vault-0.5.0


## - Check the pods and you should see a vault injector pod running.
⇒  kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
devwebapp-56c46857c4-wpxj6             1/1     Running   0          4m7s
vault-agent-injector-9bcc59498-962vp   1/1     Running   0          23s
```


## - Create a patch for the app deployed
- The patch has annotations which will instruct vault-injector to add sidecars to the deployment. 
- (Init side-car and vault-agent side-car)
```
⇒  cat 04-deploy-devweb-app-patch.yaml
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "devweb-app"
        vault.hashicorp.com/agent-inject-secret-credentials.txt: "secret/data/devwebapp/config"

-- Create kv secret for this app
⇒  vault kv put secret/devwebapp/config username='girraaafff' password='saaalsaa'
Success! Data written to: secret/devwebapp/config


-- Create a Vault policy for this app
⇒  cat devweb-app-policy.hcl
path "secret/devwebapp/config" {
  capabilities = ["read"]
}

⇒  vault policy write devweb-app devweb-app-policy.hcl
Success! Uploaded policy: devweb-app

-- Create a k8-auth role in vault for this app
⇒  vault write auth/kubernetes/role/devweb-app \
        bound_service_account_names=internal-app \
        bound_service_account_namespaces=default \
        policies=devweb-app \
        ttl=24h
Success! Data written to: auth/kubernetes/role/devweb-app


-- Deploy the patch
⇒  kubectl patch deployment devwebapp --patch "$(cat yaml-files/04-deploy-devweb-app-patch.yaml)"
deployment.apps/devwebapp patched

⇒  kubectl get pods
NAME                                   READY   STATUS     RESTARTS   AGE
devwebapp-56c46857c4-wpxj6             1/1     Running    0          6m11s
devwebapp-75bd768987-tn98w             0/2     Init:0/1   0          21s
vault-agent-injector-9bcc59498-962vp   1/1     Running    0          2m27s


```


# -- Validate the patch deployed (both app & vault-agent side-car)
-- Check the pods, you should see a new pod initialized and deployed with patch. 
-- Check the READY status, you will see two container starting, Vault injector added vault-agent sidecar
```
-- Check the number of containers in devwebapp pod (2/2 READY)
⇒  kubectl get pods
NAME                                   READY   STATUS    RESTARTS   AGE
devwebapp-58c4895874-tgtr2             2/2     Running   0          5m53s
vault-agent-injector-9bcc59498-kvpvf   1/1     Running   0          27m

```


# - Let's check if the deployment pulled secrets from Vault
```
⇒  kubectl exec -it devwebapp-75bd768987-xrnk4 -c app -- cat /vault/secrets/credentials.txt
password: salsa
username: giraffe
```

# - Some helm commands 
```
⇒  helm list
NAME 	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART      	APP VERSION
vault	default  	1       	2020-06-19 03:56:45.219072 -0400 EDT	deployed	vault-0.4.0

Uninstall Vault k8 helm chart
⇒  helm uninstall vault
release "vault" uninstalled

You should see the vault-injector or vault-o pods are terminated
⇒  kubectl get pods
No resources found.
```


# -- Troubleshooting & Checks
```
⇒  kubectl logs devwebapp-75bd768987-xrnk4 -c vault-agent-init -f
==> Vault server started! Log data will stream in below:

==> Vault agent configuration:

                     Cgo: disabled
               Log Level: info
                 Version: Vault v1.3.2

2020-06-19T08:10:54.478Z [INFO]  sink.file: creating file sink
2020-06-19T08:10:54.478Z [INFO]  sink.file: file sink configured: path=/home/vault/.token mode=-rw-r-----
2020-06-19T08:10:54.479Z [INFO]  template.server: starting template server
2020/06/19 08:10:54.479148 [INFO] (runner) creating new runner (dry: false, once: false)
2020/06/19 08:10:54.483911 [INFO] (runner) creating watcher
2020-06-19T08:10:54.484Z [INFO]  auth.handler: starting auth handler
2020-06-19T08:10:54.484Z [INFO]  auth.handler: authenticating
2020-06-19T08:10:54.485Z [INFO]  sink.server: starting sink server
2020-06-19T08:10:54.518Z [INFO]  auth.handler: authentication successful, sending token to sinks
2020-06-19T08:10:54.518Z [INFO]  auth.handler: starting renewal process
2020-06-19T08:10:54.543Z [INFO]  template.server: template server received new token
2020/06/19 08:10:54.552480 [INFO] (runner) stopping
2020/06/19 08:10:54.552518 [INFO] (runner) creating new runner (dry: false, once: false)
2020/06/19 08:10:54.552604 [INFO] (runner) creating watcher
2020/06/19 08:10:54.552655 [INFO] (runner) starting
2020-06-19T08:10:54.587Z [INFO]  sink.file: token written: path=/home/vault/.token
2020-06-19T08:10:54.587Z [INFO]  sink.server: sink server stopped
2020-06-19T08:10:54.587Z [INFO]  sinks finished, exiting
2020-06-19T08:10:54.591Z [INFO]  auth.handler: renewed auth token
2020/06/19 08:10:54.681223 [INFO] (runner) rendered "(dynamic)" => "/vault/secrets/credentials.txt"
2020/06/19 08:10:54.681247 [INFO] (runner) stopping
2020-06-19T08:10:54.681Z [INFO]  template.server: template server stopped
```


