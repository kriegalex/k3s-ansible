# Actual Budget Ansible Role

This Ansible role deploys Actual Budget on a k3s cluster using Helm.

## Requirements

- k3s cluster
- Helm 3.x
- kubectl configured with cluster access
- Python packages: kubernetes, openshift

## Role Variables

See `defaults/main.yml` for all available variables and their default values.

Key variables you might want to override:

```yaml
actual_budget_namespace: "actual-budget"
actual_budget_ingress_enabled: false
actual_budget_ingress_host: "budget.local"
actual_budget_storage_size: "1Gi"
