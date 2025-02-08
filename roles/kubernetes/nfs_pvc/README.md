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

## Example Usage in Deployments

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