# Longhorn

## Prerequisite

Make sure a bucket named longhorn (see defaults/main.yml) exists in the S3 object storage

## Uninstall

To prevent Longhorn from being accidentally uninstalled (which leads to data lost), we introduce a new setting, deleting-confirmation-flag. If this flag is false, the Longhorn uninstallation job will fail. Set this flag to true to allow Longhorn uninstallation. You can set this flag using setting page in Longhorn UI or `kubectl -n longhorn-system patch -p '{"value": "true"}' --type=merge lhs deleting-confirmation-flag`

To prevent damage to the Kubernetes cluster, we recommend deleting all Kubernetes workloads using Longhorn volumes (PersistentVolume, PersistentVolumeClaim, StorageClass, Deployment, StatefulSet, DaemonSet, etc).