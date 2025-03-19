# Ansible Role: n8n Helm Chart

This Ansible role installs and configures the n8n Helm chart on a Kubernetes cluster. n8n is a fair-code workflow automation platform with native AI capabilities.

## Requirements

- Kubernetes cluster
- Helm v3.x installed on the control node
- kubectl configured to connect to your cluster
- Access to the Kubernetes cluster from the Ansible controller

## Role Variables

See defaults/main.yml for a complete list of available variables.

### Helm Chart Configuration

```yaml
n8n_helm_chart_name: "n8n"
n8n_helm_repo_name: "community-charts"
n8n_helm_repo_url: "https://community-charts.github.io/helm-charts"
n8n_helm_chart_version: "1.3.5"
n8n_helm_release_name: "n8n"
n8n_namespace: "n8n"
n8n_create_namespace: true
n8n_log_level: "info"
n8n_timezone: "UTC"
```

### Database Configuration

```yaml
# Database type: sqlite or postgresdb
n8n_db_type: "sqlite"
n8n_table_prefix: ""

# PostgreSQL configuration (internal via Bitnami chart)
n8n_db_type: "postgresdb"
n8n_postgresql_enabled: true

# PostgreSQL configuration (external)
n8n_db_type: "postgresdb"
n8n_use_external_postgresql: true
n8n_external_postgresql:
  host: "postgresql.example.com"
  port: 5432
  username: "postgres"
  password: "secretpassword"
  database: "n8n"
  existing_secret: ""
```

### Redis Configuration (for queue mode)

```yaml
# Internal Redis via Bitnami chart
n8n_redis_enabled: true

# External Redis
n8n_use_external_redis: true
n8n_external_redis:
  host: "redis.example.com"
  port: 6379
  username: "default"
  password: "secretpassword"
  existing_secret: ""
```

### Worker and Webhook Configuration

```yaml
# Worker settings
n8n_worker_mode: "queue"  # Options: regular (single-node), queue
n8n_worker_concurrency: 10
n8n_worker_count: 2
n8n_worker_autoscaling_enabled: true

# Webhook settings
n8n_webhook_mode: "regular"
n8n_webhook_url: ""
n8n_webhook_count: 2
n8n_webhook_autoscaling_enabled: false
```

### Resource Configuration

```yaml
n8n_main_resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "512m"
    memory: "512Mi"

n8n_worker_resources:
  requests:
    cpu: "1000m"
    memory: "250Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"
```

### Ingress Configuration

```yaml
n8n_ingress_enabled: true
n8n_ingress_host: "n8n.example.com"
n8n_ingress_path: "/"
n8n_ingress_path_type: "Prefix"
n8n_ingress_annotations:
  kubernetes.io/ingress.class: "nginx"
  cert-manager.io/cluster-issuer: "letsencrypt-prod"
n8n_ingress_tls_enabled: true
n8n_ingress_tls_secret_name: "n8n-tls-secret"
```

## Example Playbook

```yaml
- hosts: localhost
  connection: local
  gather_facts: true
  roles:
    - role: kubernetes.n8n
      vars:
        n8n_helm_release_name: "n8n"
        n8n_namespace: "n8n"
        n8n_ingress_enabled: true
        n8n_ingress_host: "n8n.example.com"
        n8n_db_type: "postgresdb"
        n8n_postgresql_enabled: true
```

## Usage Example

Here's an example of how to use this role in a playbook:

```yaml
- hosts: localhost
  connection: local
  gather_facts: true
  vars:
    n8n_helm_release_name: "n8n"
    n8n_namespace: "n8n"
    n8n_create_namespace: true
    n8n_log_level: "warn"
    
    # Database config
    n8n_db_type: "postgresdb"
    n8n_use_external_postgresql: true
    n8n_external_postgresql:
      host: "postgresql.example.com"
      port: 5432
      username: "n8nuser"
      password: "securepassword"
      database: "n8n"
    
    # Queue mode with Redis
    n8n_worker_mode: "queue"
    n8n_use_external_redis: true
    n8n_external_redis:
      host: "redis.example.com"
      port: 6379
      password: "redispassword"
    
    # Ingress configuration
    n8n_ingress_enabled: true
    n8n_ingress_host: "n8n.example.com"
    n8n_ingress_annotations:
      kubernetes.io/ingress.class: "nginx"
      cert-manager.io/cluster-issuer: "letsencrypt-prod"
    n8n_ingress_tls_enabled: true
    n8n_ingress_tls_secret_name: "n8n-tls-secret"
    
    # Resources
    n8n_main_resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "512m"
        memory: "512Mi"
    
    n8n_worker_resources:
      requests:
        cpu: "1000m"
        memory: "250Mi"
      limits:
        cpu: "2000m"
        memory: "2Gi"
  
  roles:
    - role: kubernetes.n8n
```

This Ansible role provides a complete deployment solution for n8n on Kubernetes, offering flexibility for different database configurations, queue modes, and deployment scenarios.

## License

MIT
