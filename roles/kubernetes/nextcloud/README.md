# Ansible Role: Nextcloud

This role deploys Nextcloud on a Kubernetes cluster using Helm.

## Prerequisites

- Kubernetes cluster
- Helm 3.x
- `kubectl` configured to access your cluster
- `ansible-vault` installed

## Required Vault Secrets

This role requires several secure values to be stored in an Ansible vault. Follow these steps to create and manage your secrets:

1. Create a new vault file:
```bash
cd ~/workspace/k3s-ansible/inventory/my-cluster
ansible-vault create group_vars/vault.yml
```

2. Edit existing vault file:
```bash
ansible-vault edit group_vars/vault.yml
```

3. Required vault variables:
```yaml
# Admin password for Nextcloud
vault_nextcloud_admin_password: "your-secure-admin-password"

# PostgreSQL database password
vault_nextcloud_db_password: "your-secure-db-password"

# Redis password
vault_nextcloud_redis_password: "your-secure-redis-password"

# Collabora admin password (if using Collabora)
vault_nextcloud_collabora_password: "your-secure-collabora-password"

# TLS Certificate and Key (if using HTTPS)
vault_nextcloud_tls_cert: |
  -----BEGIN CERTIFICATE-----
  Your certificate here
  -----END CERTIFICATE-----

vault_nextcloud_tls_key: |
  -----BEGIN PRIVATE KEY-----
  Your private key here
  -----END PRIVATE KEY-----
```

### Generating Secure Passwords

You can generate secure passwords using these commands:

```bash
# Generate random 32-character password
openssl rand -base64 24

# Alternative using /dev/urandom
< /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-32};echo;
```

### Managing TLS Certificates

If you're using cert-manager or Let's Encrypt:
```bash
# View your certificate
kubectl get secret nextcloud-tls -n nextcloud -o jsonpath='{.data.tls\.crt}' | base64 -d

# View your private key
kubectl get secret nextcloud-tls -n nextcloud -o jsonpath='{.data.tls\.key}' | base64 -d
```

If you're using self-signed certificates:
```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout nextcloud.key \
  -out nextcloud.crt \
  -subj "/CN=nextcloud.kube.home"

# Convert to base64 and add to vault
cat nextcloud.crt | base64 -w 0  # Use this for vault_nextcloud_tls_cert
cat nextcloud.key | base64 -w 0  # Use this for vault_nextcloud_tls_key
```

## Usage

1. Create your vault file with the required secrets:
```bash
ansible-vault create group_vars/vault.yml
```

2. Update your playbook variables in `group_vars/kubernetes.yml`:
```yaml
nextcloud:
  host: nextcloud.kube.home
  admin_user: admin
  
  ingress:
    enabled: true
    className: nginx
    tls:
      enabled: true
      secretName: nextcloud-tls
      hosts:
        - nextcloud.kube.home

  persistence:
    enabled: true
    storageClass: "local-path"
    size: 8Gi

  postgresql:
    enabled: true
    global:
      postgresql:
        auth:
          username: nextcloud
          database: nextcloud

  redis:
    enabled: true
```

3. Run your playbook with vault password:
```bash
# Using vault password file
ansible-playbook -i inventory playbook.yml --vault-password-file ~/.vault_pass

# Or prompting for vault password
ansible-playbook -i inventory playbook.yml --ask-vault-pass
```

## Vault Password Management

### Creating a vault password file
```bash
# Generate random password
openssl rand -base64 32 > ~/.vault_pass
chmod 600 ~/.vault_pass

# Use this password to encrypt your vault
ansible-vault create --vault-password-file ~/.vault_pass group_vars/vault.yml
```

### Rotating vault passwords
```bash
# Rekey existing vault with new password
ansible-vault rekey --vault-password-file ~/.vault_pass.old --new-vault-password-file ~/.vault_pass.new group_vars/vault.yml
```

## Security Considerations

1. Never commit unencrypted vault files to version control
2. Store vault passwords securely and separately from your playbooks
3. Rotate passwords periodically
4. Use strong passwords (minimum 32 characters recommended)
5. Restrict vault file permissions: `chmod 600 group_vars/vault.yml`

## Troubleshooting

If you need to view the contents of your vault:
```bash
# View encrypted vault
ansible-vault view group_vars/vault.yml

# Decrypt vault (temporary)
ansible-vault decrypt group_vars/vault.yml
# Don't forget to re-encrypt after viewing!
ansible-vault encrypt group_vars/vault.yml
```
