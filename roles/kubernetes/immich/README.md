# Immich

## Backup & Restore

Always refer to the instructions here as a baseline:

- https://immich.app/docs/administration/backup-and-restore

### Note on migration

- https://github.com/immich-app/immich/discussions/9060

Immich currently requires `pgdump_all` instead of `pg_dump` for the migration process to work. Unlike paperless, nextcloud or gitea, immich cannot use the Cluster import bootstrap functionality. For now, a manual backup and a manual restore is the easiest and quickest option. Simply set immich to skip the database migrations when doing migrations (see ansible variables).

### Manual backup

```bash
kubectl exec immich-db-1 -c postgres -- sh -c 'pg_dumpall --clean --if-exists --username=postgres | gzip > "/var/lib/postgresql/data/dump.sql.gz"'
# change the namespace to suit your needs
kubectl -n namespace cp -c postgres immich-db-1:/var/lib/postgresql/data/dump.sql.gz ./dump.sql.gz 
```

### Manual restore

```bash
gunzip < "./dump.sql.gz" | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" > dump.sql
# change the namespace to suit your needs
kubectl -n namespace cp -c postgres ./dump.sql immich-db-1:/var/lib/postgresql/data/dump.sql
# avoid passwords with "$" if possible
kubectl -n namespace exec immich-db-1 -c postgres -- sh -c 'PGPASSWORD="YOUR_PASSWORD" psql --dbname=postgres --username=immich --host=localhost -f /var/lib/postgresql/data/dump.sql'
kubectl -n namespace exec immich-db-1 -c postgres -- rm /var/lib/postgresql/data/dump.sql
```