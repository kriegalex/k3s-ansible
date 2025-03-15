# Bitcoin Core Role

This role deploys Bitcoin Core on Kubernetes using Helm.

## Prerequisites

- Working Kubernetes cluster
- NFS server for persistent storage (recommended)
- Helm installed

## Configuration

Create your own configuration by copying the example:

```bash
cp vars/main.yml.example vars/main.yml
```

Then edit the `vars/main.yml` file to customize your deployment.

## Storage Options

### NFS Storage (Recommended)
For persistent blockchain storage, NFS is recommended:

```yaml
bitcoin_persistence_enabled: true
bitcoin_persistence_nfs_claim_name: "nfs-bitcoin"
bitcoin_persistence_nfs_server: "10.0.0.2"
bitcoin_persistence_nfs_path: "/mnt/user/bitcoin"
```

### Alternative Storage
You can use any storage class available in your cluster by setting:

```yaml
bitcoin_persistence_enabled: true
bitcoin_persistence_storage_class: "longhorn"  # Your storage class
bitcoin_persistence_nfs_claim_name: ""  # Leave empty to skip NFS setup
```

Or use an existing claim:

```yaml
bitcoin_persistence_enabled: true
bitcoin_persistence_existing_claim: "your-existing-pvc"  # Takes precedence
```

## Deployment

Include this role in your playbook:

```yaml
- hosts: k3s_master
  roles:
    - role: kubernetes/bitcoin
      when: bitcoin_enabled
```

## Important Notes

- Bitcoin blockchain requires significant storage (500GB+ for full node)
- Consider enabling pruning mode if storage is limited
- Secure RPC credentials in vault or secure variables
- Initial blockchain sync can take several days
