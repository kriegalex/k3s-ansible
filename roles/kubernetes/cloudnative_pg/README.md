# Ansible Role: CloudNativePG

## What is CloudNativePG?
CloudNativePG is a Kubernetes operator that manages PostgreSQL databases natively in Kubernetes. It's designed for:
- Running PostgreSQL databases directly in your Kubernetes cluster
- Providing high availability with primary/standby architecture
- Automating database operations (backups, failovers, updates)

## When Should You Use This Role?

### ✅ Use CloudNativePG when:
- Your applications (like Nextcloud, Immich) are running inside the same Kubernetes cluster
- You need automated high availability for your PostgreSQL databases
- You want native Kubernetes integration for your databases
- You're comfortable managing Kubernetes resources

### ❌ Maybe Skip CloudNativePG when:
- Your applications run outside Kubernetes (on regular VMs or bare metal)
- You need a simpler single-instance database setup
- You're new to Kubernetes and want to start with basics
- You prefer traditional database management

## Comparison with Bitnami PostgreSQL

| Feature | CloudNativePG | Bitnami PostgreSQL |
|---------|---------------|-------------------|
| Complexity | Higher | Lower |
| Kubernetes Native | Yes | No (basic container) |
| High Availability | Built-in | Manual setup |
| Resource Usage | Higher (needs multiple pods) | Lower |
| Learning Curve | Steeper | Gentler |

## Prerequisites
- A working Kubernetes cluster
- Kubernetes experience
- Storage class for persistent volumes
- Basic PostgreSQL knowledge

## Related Application Considerations
When using applications like:
- **Nextcloud**: Choose CloudNativePG if running Nextcloud in Kubernetes
- **Immich**: Choose CloudNativePG for Immich charts >= 0.9.0
- **Other apps**: Consider your deployment environment and HA requirements

## Memory Requirements
Minimum recommended:
- 2GB RAM for basic setup
- Additional memory for each standby instance
- Extra resources for Kubernetes overhead

## Storage Requirements
- Minimum 1GB for system database
- Plan additional storage based on your application needs
- Backup storage consideration needed