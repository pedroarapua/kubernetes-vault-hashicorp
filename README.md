## Requirements ##
- Docker
- Kind
- Helm
- Vault (https://learn.hashicorp.com/tutorials/vault/getting-started-install)

## Create Cluster Kubernetes ##

kind create cluster --name kms

## Install Vault ##

helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault --set "server.dev.enabled=true"

## Set a secret in Vault ##

kubectl exec -it vault-0 -- /bin/sh
# Enable key value configuration #
vault secrets enable -path=internal kv-v2
# Set key value configuration #
vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"

## Configure Kubernetes Authentication ##

kubectl exec -it vault-0 -- /bin/sh
# Enable Kubernetes Auth #
vault auth enable kubernetes
# Set KUbernetes Auth Configuration #
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    issuer="https://kubernetes.default.svc.cluster.local"
# Create Policy #
vault policy write internal-app - <<EOF
path "internal/data/database/config" {
  capabilities = ["read"]
}
EOF
# Create Kubernetes Auth Role #
vault write auth/kubernetes/role/internal-app \
    bound_service_account_names=internal-app \
    bound_service_account_namespaces=default \
    policies=internal-app \
    ttl=24h
# Create Serivce Account in Kubernetes #
kubectl create sa internal-app

## Deployment App ##

# Deploy without annotations #
kubectl apply --filename deployment.yaml

# Patch deployment with annotations #
kubectl patch deployment orgchart --patch "$(cat patch.yaml)"

## Optional Commands ##

# PortForward Pod Vault #
kubectl port-forward vault-0 8200:8200

# Enter Method and Token #
http://localhost:8200/ui/vault/auth?with=token (field method: token, field token: root)