# Vault Variables to Kubernetes Secrets Migration Guide

This guide helps you migrate from the removed Ansible vault variables to Kubernetes secrets for manual helm chart deployments.

## Overview

The following vault variables have been removed from this repository as it now focuses solely on k3s cluster provisioning. If you were using the app installation roles, you can manually create Kubernetes secrets and deploy apps using helm charts directly.

## Removed Vault Variables

The following variables were removed from `inventory/sample/group_vars/all/vault.yml`:

- Nextcloud: `vault_nextcloud_admin_password`, `vault_nextcloud_db_password`, `vault_nextcloud_redis_password`, `vault_nextcloud_collabora_password`
- Immich: `vault_immich_user_password`
- Paperless: `vault_paperless_db_password`, `vault_paperless_redis_password`, `vault_paperless_secret_key`, `vault_paperless_admin_password`
- Gitea: `vault_gitea_redis_password`, `vault_gitea_db_password`, `vault_gitea_admin_password`
- qBittorrent: `vault_qbittorrent_vpn_pia_password`
- Grafana: `vault_grafana_admin_password`
- CloudNativePG: `vault_cnpg_backup_s3_access_key`, `vault_cnpg_backup_s3_secret_key`

## Migration Commands

### Nextcloud

```bash
# Create namespace
kubectl create namespace nextcloud

# Create admin password secret
kubectl create secret generic nextcloud-admin \
  --namespace=nextcloud \
  --from-literal=password='YOUR_ADMIN_PASSWORD'

# Create PostgreSQL password secret
kubectl create secret generic nextcloud-db \
  --namespace=nextcloud \
  --from-literal=password='YOUR_DB_PASSWORD'

# Create Redis password secret
kubectl create secret generic nextcloud-redis \
  --namespace=nextcloud \
  --from-literal=password='YOUR_REDIS_PASSWORD'

# Create Collabora password secret (if using)
kubectl create secret generic nextcloud-collabora \
  --namespace=nextcloud \
  --from-literal=password='YOUR_COLLABORA_PASSWORD'
```

**Helm values reference:**
```yaml
nextcloud:
  password:
    secretName: nextcloud-admin
    secretKey: password

postgresql:
  auth:
    existingSecret: nextcloud-db

redis:
  auth:
    existingSecret: nextcloud-redis
```

---

### Immich

```bash
# Create namespace
kubectl create namespace immich

# Create PostgreSQL user password secret
kubectl create secret generic immich-db \
  --namespace=immich \
  --from-literal=password='YOUR_IMMICH_DB_PASSWORD'
```

**Helm values reference:**
```yaml
postgresql:
  auth:
    password: # Reference from immich-db secret
    existingSecret: immich-db
```

---

### Paperless-ngx

```bash
# Create namespace
kubectl create namespace paperless

# Create PostgreSQL password secret
kubectl create secret generic paperless-db \
  --namespace=paperless \
  --from-literal=password='YOUR_DB_PASSWORD'

# Create Redis password secret
kubectl create secret generic paperless-redis \
  --namespace=paperless \
  --from-literal=password='YOUR_REDIS_PASSWORD'

# Create Paperless secret key
kubectl create secret generic paperless-secret \
  --namespace=paperless \
  --from-literal=secret-key='YOUR_SECRET_KEY'

# Create admin password secret
kubectl create secret generic paperless-admin \
  --namespace=paperless \
  --from-literal=password='YOUR_ADMIN_PASSWORD'
```

**Helm values reference:**
```yaml
paperless:
  env:
    PAPERLESS_SECRET_KEY:
      valueFrom:
        secretKeyRef:
          name: paperless-secret
          key: secret-key
    PAPERLESS_ADMIN_PASSWORD:
      valueFrom:
        secretKeyRef:
          name: paperless-admin
          key: password

postgresql:
  auth:
    existingSecret: paperless-db

redis:
  auth:
    existingSecret: paperless-redis
```

---

### Gitea

```bash
# Create namespace
kubectl create namespace gitea

# Create Redis password secret
kubectl create secret generic gitea-redis \
  --namespace=gitea \
  --from-literal=password='YOUR_REDIS_PASSWORD'

# Create PostgreSQL password secret
kubectl create secret generic gitea-db \
  --namespace=gitea \
  --from-literal=password='YOUR_DB_PASSWORD'

# Create admin user secret
kubectl create secret generic gitea-admin \
  --namespace=gitea \
  --from-literal=username='gitea_admin' \
  --from-literal=password='YOUR_ADMIN_PASSWORD' \
  --from-literal=email='admin@example.com'
```

**Helm values reference:**
```yaml
gitea:
  admin:
    existingSecret: gitea-admin

postgresql:
  auth:
    existingSecret: gitea-db

redis:
  auth:
    existingSecret: gitea-redis
```

---

### qBittorrent (with VPN)

```bash
# Create namespace
kubectl create namespace qbittorrent

# Create VPN credentials secret (for PIA)
kubectl create secret generic qbittorrent-vpn \
  --namespace=qbittorrent \
  --from-literal=username='YOUR_PIA_USERNAME' \
  --from-literal=password='YOUR_PIA_PASSWORD'
```

**Helm values reference:**
```yaml
env:
  VPN_SERVICE_PROVIDER: "private internet access"
  VPN_TYPE: "openvpn"
  OPENVPN_USER:
    valueFrom:
      secretKeyRef:
        name: qbittorrent-vpn
        key: username
  OPENVPN_PASSWORD:
    valueFrom:
      secretKeyRef:
        name: qbittorrent-vpn
        key: password
```

---

### Grafana (Prometheus Stack)

```bash
# Create namespace
kubectl create namespace monitoring

# Create Grafana admin password secret
kubectl create secret generic grafana-admin \
  --namespace=monitoring \
  --from-literal=admin-password='YOUR_ADMIN_PASSWORD' \
  --from-literal=admin-user='admin'
```

**Helm values reference (kube-prometheus-stack):**
```yaml
grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
```

---

### CloudNativePG S3 Backup

```bash
# Create namespace (or use existing app namespace)
kubectl create namespace postgresql

# Create S3 backup credentials secret
kubectl create secret generic cnpg-s3-backup \
  --namespace=postgresql \
  --from-literal=ACCESS_KEY_ID='YOUR_ACCESS_KEY' \
  --from-literal=ACCESS_SECRET_KEY='YOUR_SECRET_KEY'
```

**CloudNativePG Cluster manifest reference:**
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-cluster
  namespace: postgresql
spec:
  backup:
    barmanObjectStore:
      destinationPath: "s3://my-bucket/backups/"
      s3Credentials:
        accessKeyId:
          name: cnpg-s3-backup
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: cnpg-s3-backup
          key: ACCESS_SECRET_KEY
```

---

## Best Practices

### 1. Use Sealed Secrets or External Secrets Operator

For production environments, consider using:

**Sealed Secrets:**
```bash
# Install sealed-secrets controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets --namespace kube-system

# Create a sealed secret
kubectl create secret generic my-secret \
  --from-literal=password='mypassword' \
  --dry-run=client -o yaml | \
  kubeseal -o yaml > my-sealed-secret.yaml
```

**External Secrets Operator:**
```bash
# Install external-secrets
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets --namespace external-secrets-system --create-namespace

# Reference secrets from external stores (AWS Secrets Manager, Vault, etc.)
```

### 2. Generate Strong Passwords

```bash
# Generate a random 32-character password
openssl rand -base64 32

# Generate alphanumeric password
tr -dc A-Za-z0-9 </dev/urandom | head -c 32; echo
```

### 3. Backup Your Secrets

```bash
# Export all secrets from a namespace (store securely)
kubectl get secrets --namespace=myapp -o yaml > secrets-backup.yaml

# Encrypt the backup
gpg --symmetric --cipher-algo AES256 secrets-backup.yaml
```

### 4. Secret Management

```bash
# View secret (base64 encoded)
kubectl get secret my-secret -o yaml

# Decode secret value
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d

# Update existing secret
kubectl create secret generic my-secret \
  --from-literal=password='new-password' \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## Migration Checklist

- [ ] Export your current Ansible vault variables
- [ ] Generate strong passwords for production use
- [ ] Create Kubernetes secrets for each application
- [ ] Update helm values files to reference the secrets
- [ ] Test deployments in a non-production environment
- [ ] Backup secret manifests securely
- [ ] Consider implementing Sealed Secrets or External Secrets Operator
- [ ] Document your secret management strategy
- [ ] Remove old Ansible vault file from version control

---

## Additional Resources

- [Kubernetes Secrets Documentation](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Helm Secrets Management](https://helm.sh/docs/howto/charts_tips_and_tricks/#using-secrets)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
- [CloudNativePG Backup Documentation](https://cloudnative-pg.io/documentation/current/backup/)

---

## Support

Since this repository now focuses solely on k3s cluster provisioning, application deployment and secret management are the responsibility of the end user. Consider:

- Using GitOps tools (ArgoCD, FluxCD) for application deployment
- Implementing a secrets management solution (Vault, External Secrets)
- Following Kubernetes security best practices
- Regular secret rotation policies
