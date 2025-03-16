# Bluesky PDS Ansible Role

This role deploys a Bluesky Personal Data Server (PDS) on a Kubernetes cluster using its Helm chart.

## Requirements

- Kubernetes cluster
- Helm installed
- kubectl configured
- Ingress controller (like nginx-ingress)
- Cert-manager for TLS (optional, but recommended)

## Overview

This role simplifies the deployment of a self-hosted Bluesky PDS on your Kubernetes cluster. It handles the installation and configuration of all necessary components.

## Installation and Configuration

### 1. Basic Configuration

Enable the role in your playbook and set the essential variables:

```yaml
- hosts: k3s_cluster
  vars:
    global_domain_name: "example.com"
    bluesky_pds_enabled: true
    bluesky_pds_subdomain: "pds"  # Your PDS will be available at pds.example.com
    bluesky_pds_admin_password: "{{ vault_bluesky_pds_admin_password }}"
    bluesky_pds_service_handle_domains: ["example.com"]  # Allowed handle domains
  roles:
    - role: kubernetes/bluesky_pds
```

### 2. DID Configuration

Every Bluesky PDS needs a Decentralized Identifier (DID). You have two options:

```yaml
# Option 1: Use a placeholder DID for initial setup (you'll need to update later)
bluesky_pds_server_did: "did:plc:temporary"

# Option 2: If you already have a DID from a previous setup
bluesky_pds_server_did: "did:plc:your-existing-did"
```

### 3. JWT and Security Configuration

For production environments, make sure to set secure values for JWT and other secrets:

```yaml
bluesky_pds_jwt_secret: "{{ vault_bluesky_pds_jwt_secret }}"
bluesky_pds_plc_rotation_key_secret: "{{ vault_bluesky_pds_plc_rotation_key_secret }}"
```

### 4. Storage Configuration

By default, the role uses persistent storage. You can customize it:

```yaml
bluesky_pds_persistence_enabled: true
bluesky_pds_persistence_storage_class: "longhorn"  # Use your preferred storage class
bluesky_pds_persistence_size: "10Gi"
```

### 5. Invite Codes

Control who can create accounts on your PDS:

```yaml
bluesky_pds_invite_required: true  # Require invite codes for registration
```

### 6. SMTP Configuration

Configure email notifications:

```yaml
bluesky_pds_smtp_enabled: true
bluesky_pds_smtp_host: "smtp.example.com"
bluesky_pds_smtp_port: 587
bluesky_pds_smtp_user: "notifications@example.com"
bluesky_pds_smtp_password: "{{ vault_bluesky_pds_smtp_password }}"
bluesky_pds_smtp_from_address: "notifications@example.com"
bluesky_pds_smtp_secure: true  # Use TLS
```

### 7. PDS Crawler Configuration

Configure the BlueSky crawler for improved federation:

```yaml
bluesky_pds_crawler_enabled: true
bluesky_pds_crawler_service_did: "did:web:bsky.network"
```

### 8. Backup Configuration

Setup automatic backups of your PDS data:

```yaml
bluesky_pds_backups_enabled: true
bluesky_pds_backups_schedule: "0 2 * * *"  # Daily at 2am
bluesky_pds_backups_retention: 7  # Keep 7 days of backups
bluesky_pds_backups_storage_class: "{{ bluesky_pds_persistence_storage_class }}"
bluesky_pds_backups_size: "5Gi"
```

### 9. Domain Verification

Configure domain verification to prevent handle spoofing:

```yaml
bluesky_pds_domain_verification_enabled: true
bluesky_pds_verification_record_name: "_atproto"
bluesky_pds_verification_record_text: "did={{ bluesky_pds_server_did }}"
```

## Post-Installation Steps

1. After deployment, access your PDS admin panel at `https://pds.example.com/admin`
2. Log in with the admin password you configured
3. Generate invite codes if you enabled that feature
4. If you used a temporary DID, the role will automatically fetch and apply the generated DID
5. DNS configuration for domain verification will be output at the end of the playbook run

## Security Considerations

- Always override the default admin password using Ansible Vault
- Configure proper ingress settings with TLS enabled
- Use invite codes to control who can join your PDS
- Set appropriate resource limits to prevent resource exhaustion
- Configure proper domain name(s) and DID for your server
- Use Ansible Vault for all sensitive values including JWT secrets and moderator passwords

## Additional Resources

For more detailed information on setting up a Bluesky PDS, refer to the [Helm chart author's guide](https://nerkho.ch/blog/self-hosted-pds-on-k8s/).

Additional documentation:
- [AT Protocol Documentation](https://atproto.com/docs)
- [Bluesky PDS API Reference](https://docs.bsky.app/)
