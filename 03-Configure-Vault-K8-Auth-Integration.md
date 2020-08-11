## Configure the Vault K8 Auth + k8 Integration 
To configure this integration, you will need
1. k8 host url
2. k8 ca cert
3. bearer token for vault_reviewer SA  


### Configuring the k8 Auth
```
#-- STEP 1
Mount the kubernetes auth backend:

⇒  vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/


⇒  vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_1ad77c83    n/a
token/         token         auth_token_7c0bb412         token based credentials
```

```
#-- STEP2
Configure the auth backend with the pulblic key of Kubernetes' JWT signing key,
the host for the Kubernetes API, and the CA cert used for the API. Depending on
your configuration, most of these values can be found through the `kubectl
config view` command. Replace the values below with the values for your system.

-- kubernetes_host can be found using the below command.
⇒  kubectl cluster-info
Kubernetes master is running at https://172.42.42.100:6443 <-- this is the k8 host url
KubeDNS is running at https://172.42.42.100:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

⇒  curl -kv https://172.42.42.100:6443/healthz ( should respond back with ok )

-- bearer token for the token reviewer SA (vault_reviewer) can be found using the below command
⇒  kubectl get secret \
  $(kubectl get serviceaccount vault-reviewer -o json | jq -r '.secrets[0].name') \
  -o json | jq -r '.data .token' | base64 -D -

-- kubernetes CA cert can be fetched using command
⇒  kubectl get serviceaccount vault-reviewer -o json | jq -r '.secrets[0].name'
⇒  kubectl get secret vault-reviewer-token-bcdzp -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo
⇒  kubectl get secret vault-reviewer-token-bcdzp -o jsonpath="{.data['ca\.crt']}" | base64 --decode > k8-ca.crt

You can check the contents of the cert using openssl command
openssl x509 -text -noout -in k8-ca.crt

⇒  export K8_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}')
⇒  export SA_JWT_TOKEN=$(kubectl get secret \
  $(kubectl get serviceaccount vault-reviewer -o json | jq -r '.secrets[0].name') \
  -o json | jq -r '.data .token' | base64 -D -)
On Mac, the cert is getting terminated when loading into an env variables. so I am using the k8-ca.crt file

# Enable and configure the Auth Method in Vault


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


# Write an example static secret

```
vault secrets enable -path=secret -version=1 kv
vault kv put secret/app1 username=app1 password=supasecr3t
vault read secret/app1

This will be used later in the demo

vault write secret/creds username=demo password=test
```  


# Create application policy for app1
```
cat <<EOF > app1-policy.hcl
  path "secret/app1" {
    capabilities = ["read", "list"]
  }
  path "database/creds/app1" {
    capabilities = ["read", "list"]
  }
EOF

⇒  vault policy write app1-policy app1-policy.hcl
Success! Uploaded policy: app1-policy

⇒  vault policy read app1-policy
path "secret/app1" {
    capabilities = ["read", "list"]
  }
  path "database/creds/app1" {
    capabilities = ["read", "list"]
  }
```


### Configuring 'app1' k8-auth Role in Vault

```
Roles are used to bind Kubernetes Service Account names and namespaces to a set
of Vault policies and token settings. 

-- create a role with the S.A. name "vault-auth" in the "default" namespace:
⇒  vault write "auth/kubernetes/role/app1-role" \
  bound_service_account_names="default,vault-auth" \
  bound_service_account_namespaces="default" \
  policies="app1-policy" ttl=1h

Success! Data written to: auth/kubernetes/role/app1-role

-- List the k8 auth roles created
⇒  vault list auth/kubernetes/role
Keys
----
app1-role

--Read the demo role to verify everything was configured properly:
⇒  vault read auth/kubernetes/role/app1-role
Key                                 Value
---                                 -----
bound_service_account_names         [default vault-auth]
bound_service_account_namespaces    [default]
policies                            [app1-policy]
token_bound_cidrs                   []
token_explicit_max_ttl              0s
token_max_ttl                       0s
token_no_default_policy             false
token_num_uses                      0
token_period                        0s
token_policies                      [app1-policy]
token_ttl                           1h
token_type                          default
ttl                                 1h
```


## Test the authentication method using CLI

```
⇒  export APP_JWT=$(kubectl get secret \
  $(kubectl get serviceaccount vault-auth -o json | jq -r '.secrets[0].name') \
  -o json | jq -r '.data .token' | base64 -D -)

⇒  env | grep APP_JWT
APP_JWT=eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1hdVVsaDZ5WlhlQUJBRXZzT3dFRmRMWHFVWW9HVXVLbWJYWFpKckFZcUUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6InZhdWx0LWF1dGgtdG9rZW4tbHJ3NnIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoidmF1bHQtYXV0aCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjNlNjQwY2QwLTdiMmMtNDFhYS1iNjM5LTdjNmFjYjdlYmYwYSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OnZhdWx0LWF1dGgifQ.Fm8If_-uP38ieo6CACGvO-PDyZR8pyBPcowXGaRe_dmsoKx3cmT0n8vkdEE6Q9idAaunmwBVGa___eakFPEfEjKZNf68OpyI5PxokPXmWc4Wi9yJNf5EXy9esl5y2v6AG9PU42UfOxJfuPH73Ea0TatjH7xQ_j5WUxX4EOyg1-v6w2dL6J0ZSyWk877jBmo26t9pgVfhZl8GBltWio3dhGHomdhteP_xRNVkKIo7wQdkBTy1rI6olSA_vzmIYtYnissRGG-nc8BtHF6dKCeHwbMerfMsoQQcTAUzbokN1jg8SCuz6rvpV905jbONWDb95nukvObi9r7lvW_bkyz3Bw


⇒  vault write "auth/kubernetes/login" role="app1-role" jwt="${APP_JWT}"
Key                                       Value
---                                       -----
token                                     s.TQXkbrNsPQuBPQCtqZcGj1tj
token_accessor                            AVlUdmvYdtHsKrdOBFJWE0Hu
token_duration                            1h
token_renewable                           true
token_policies                            ["app1-policy" "default"]
identity_policies                         []
policies                                  ["app1-policy" "default"]
token_meta_service_account_uid            3757f1f3-b119-4878-b7b3-31f1e5c8b1df
token_meta_role                           app1-role
token_meta_service_account_name           vault-auth
token_meta_service_account_namespace      default
token_meta_service_account_secret_name    vault-aut
```


## -- TROUBLESHOOTING
```
If this test FAILS, run api calls directly against the k8 service. The below API call is what Vault internally is using
If you don;t see thee o/p as below for the curl command, this means you have either cert issue, token issue or network issue
⇒  cat payload.json
{
  "kind": "TokenReview",
  "apiVersion": "authentication.k8s.io/v1",
  "spec": {
    "token": "COPY HERE THE BEARER TOKEN OF THE SA vault-auth"
  }
}

⇒  export K8_HOST=$(kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.server}')
⇒  export SA_JWT_TOKEN=$(kubectl get secret \
  $(kubectl get serviceaccount vault-reviewer -o json | jq -r '.secrets[0].name') \
  -o json | jq -r '.data .token' | base64 -D -)
  
  
$ curl -k -X POST --data @payload.json -H "Authorization: Bearer $SA_JWT_TOKEN" -H 'Accept: application/json' -H 'Content-Type: application/json' $KUBE_HOST/apis/authentication.k8s.io/v1/tokenreviews
{
  "kind": "TokenReview",
  "apiVersion": "authentication.k8s.io/v1",
  "metadata": {
    "creationTimestamp": null,
    "managedFields": [
      {
        "manager": "curl",
        "operation": "Update",
        "apiVersion": "authentication.k8s.io/v1",
        "time": "2020-06-19T03:48:30Z",
        "fieldsType": "FieldsV1",
        "fieldsV1": {"f:spec":{"f:token":{}}}
      }
    ]
  },
  "spec": {
    "token": "ssss"
  },
  "status": {
    "authenticated": true,
    "user": {
      "username": "system:serviceaccount:default:vault-auth",
      "uid": "3757f1f3-b119-4878-b7b3-31f1e5c8b1df",
      "groups": [
        "system:serviceaccounts",
        "system:serviceaccounts:default",
        "system:authenticated"
      ]
    }
  }
}
```
