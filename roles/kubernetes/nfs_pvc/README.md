# roles/k3s-nfs/README.md
# K3s NFS Storage Role

This role configures NFS-based persistent storage in a k3s cluster.

## Prerequisites

- Working k3s cluster
- NFS server (10.0.0.2) with shares mounted under /mnt/user/
- NFS client utilities installed on all k3s nodes

## Usage

Include this role in your playbook:

```yaml
- hosts: k3s_master
  roles:
    - k3s-nfs
```

## Available Storage

The following PVs and PVCs will be created in the 'storage' namespace:

- nfs-movies (4Ti)
- nfs-tv (4Ti)
- nfs-anime (2Ti)
- nfs-torrent (500Gi)
- nfs-nextcloud (1Ti)
- nfs-backup (2Ti)
- nfs-immich (1Ti)

## Access Modes

The following access modes are supported:
- ReadWriteOnce (default) - Volume can be mounted as read-write by a single node
- ReadWriteMany - Volume can be mounted as read-write by many nodes
- ReadOnlyMany - Volume can be mounted as read-only by many nodes

To create a read-only PVC, specify `access_mode: "ReadOnlyMany"` in your PVC definition.

## Example Usage in Deployments

### Standard PVC:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: media-pod
spec:
  containers:
  - name: media-container
    image: your-image
    volumeMounts:
    - mountPath: /movies
      name: movies-volume
  volumes:
  - name: movies-volume
    persistentVolumeClaim:
      claimName: nfs-movies
```

### Read-only PVC:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: read-only-media-pod
spec:
  containers:
  - name: media-viewer
    image: your-image
    volumeMounts:
    - mountPath: /archive
      name: archive-volume
      readOnly: true  # Explicitly set to readOnly for extra security
  volumes:
  - name: archive-volume
    persistentVolumeClaim:
      claimName: read_only_share  # PVC with ReadOnlyMany access mode
```