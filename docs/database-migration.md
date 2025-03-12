# Database Migration and NFS Integration Guide

This document explains how to use database migration for CloudNativePG and NFS persistent storage in the Ansible roles.

## Database Migration

CloudNativePG supports migrating data from one cluster to another. This is useful when:
- Upgrading PostgreSQL versions
- Moving data between clusters
- Recovering from backups in a new cluster

### How to Migrate a Database

1. Configure your role variables in `group_vars` or `host_vars`:

For **Nextcloud**:
```yaml
nextcloud_enabled: true
nextcloud_postgresql_source: "cloudnativepg"
nextcloud_postgresql_create_cluster: true

# Migration settings
nextcloud_migrate_database: true
nextcloud_migrate_source_namespace: default  # Namespace of the source database
nextcloud_migrate_source_db_name: nextcloud-db-old  # Source cluster name
```

For **Immich**:
```yaml
immich_enabled: true
immich_postgresql_source: "cloudnativepg"
immich_postgresql_create_cluster: true

# Migration settings
immich_migrate_database: true
immich_migrate_source_namespace: default
immich_migrate_source_db_name: immich-db-old
```

For **Paperless**:
```yaml
paperless_enabled: true
paperless_postgresql_source: "cloudnativepg"
paperless_postgresql_create_cluster: true

# Migration settings
paperless_migrate_database: true
paperless_migrate_source_namespace: default
paperless_migrate_source_db_name: paperless-db-old
```

For **Gitea**:
```yaml
gitea_enabled: true
gitea_postgresql_source: "cloudnativepg"
gitea_postgresql_create_cluster: true

# Migration settings
gitea_migrate_database: true
gitea_migrate_source_namespace: default
gitea_migrate_source_db_name: gitea-db-old
```

2. Run the appropriate playbook:
```bash
ansible-playbook playbooks/base_apps.yml -t nextcloud,immich,paperless,gitea
```

## NFS Integration

To use NFS storage through the `nfs_pvc` role, set the NFS claim name variables in your configuration:

For **Nextcloud**:
```yaml
nextcloud_persistence_data_enabled: true
nextcloud_persistence_data_nfs_claim_name: "nfs-nextcloud"  # This triggers NFS PVC creation
nextcloud_persistence_data_size: "50Gi"  # Specify the size
```

For **Immich**:
```yaml
immich_persistence_type: "pvc"
immich_persistence_nfs_claim_name: "nfs-immich"  # This triggers NFS PVC creation
immich_persistence_storage_size: "100Gi"  # Specify the size

# External library (e.g., for read-only access to Nextcloud photos)
immich_persistence_external_enabled: true
immich_persistence_external_nfs_claim_name: "nfs-nextcloud-ro"  # This triggers NFS PVC creation
immich_persistence_external_readonly: true
immich_persistence_external_storage_size: "100Gi"  # Specify the size
```

For **Paperless**:
```yaml
paperless_persistence_media_enabled: true
paperless_media_nfs_claim_name: "nfs-paperless-media"  # This triggers NFS PVC creation
paperless_media_storage_size: "100Gi"  # Specify the size

paperless_persistence_consume_enabled: true
paperless_consume_nfs_claim_name: "nfs-paperless-consume"  # This triggers NFS PVC creation
paperless_consume_storage_size: "10Gi"  # Specify the size

paperless_persistence_export_enabled: true
paperless_export_nfs_claim_name: "nfs-paperless-export"  # This triggers NFS PVC creation
paperless_export_storage_size: "10Gi"  # Specify the size
```

For **Gitea**:
```yaml
gitea_persistence_enabled: true
gitea_persistence_nfs_claim_name: "nfs-gitea"  # This triggers NFS PVC creation
gitea_persistence_size: "10Gi"  # Specify the size
gitea_persistence_access_mode: "ReadWriteMany"
```

Each role also supports using existing claims by setting the regular claim name variables instead:

```yaml
# Example for Nextcloud using an existing claim
nextcloud_persistence_data_enabled: true
nextcloud_persistence_data_claim_name: "my-existing-claim"  # Takes precedence over NFS claim
```

The `nfs_pvc` role will automatically create the necessary NFS persistent volumes and claims in Kubernetes based on the NFS claim name settings.
