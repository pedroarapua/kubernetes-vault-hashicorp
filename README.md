## Requirements
- Docker
- Kind
- Helm
- Vault (https://learn.hashicorp.com/tutorials/vault/getting-started-install)

### Create Kind Cluster Kubernetes
```bash
kind create cluster --name kms
```

### Install Vault Hashicorp

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault --set "server.dev.enabled=true"
```

### Config Vault Hashicorp

##### Set a secret in Vault
```bash
kubectl exec -it vault-0 -- /bin/sh
```
##### Enable key value configuration
```bash
vault secrets enable -path=internal kv-v2
```
##### Set key value configuration
```bash
vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"
echo "abc" > tmp/a1.pem
vault kv put internal/certificates/config cnpj1=@tmp/a1.pem
```

### Config Kubernetes Authentication
##### Connect vault pod
```bash
kubectl exec -it vault-0 -- /bin/sh
```
##### Enable Kubernetes Auth
```bash
vault auth enable kubernetes
```
##### Set Kubernetes Auth Configuration
```bash
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    issuer="https://kubernetes.default.svc.cluster.local"
```
##### Create Policy
```bash
vault policy write internal-app - <<EOF
path "internal/data/database/config" {
  capabilities = ["read"]
}
path "internal/data/certificates/config" {
  capabilities = ["read"]
}
EOF
```
##### Create Kubernetes Auth Role
```bash
vault write auth/kubernetes/role/internal-app \
    bound_service_account_names=internal-app \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=24h
```
##### Create Serivce Account in Kubernetes
```bash
kubectl create sa internal-app
```

### Deployment App ###

##### Deploy without annotations
```bash
kubectl apply --filename deployment.yaml
```

##### Patch deployment with annotations
```bash
kubectl patch deployment orgchart --patch "$(cat patch.yaml)"
```

### Optional Commands

##### PortForward Pod Vault
```bash
kubectl port-forward vault-0 8200:8200
```

##### Enter Method and Toke
http://localhost:8200/ui/vault/auth?with=token (field method: token, field token: root)