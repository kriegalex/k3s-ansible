# Testing Guide for k3s-ansible

This document provides instructions for testing k3s-ansible after the infrastructure refactoring.

## Quick Start

```bash
# Install testing dependencies
pipx install molecule ansible-lint

# Run default scenario (bare cluster)
cd /path/to/k3s-ansible
molecule test

# Run specific scenario
molecule test -s calico
molecule test -s cilium
molecule test -s single_node
```

## Updated Molecule Scenarios

All existing Molecule scenarios have been updated to test **bare cluster deployment** (no infrastructure components).

### Available Scenarios

| Scenario | Purpose | CNI | Infrastructure |
|----------|---------|-----|----------------|
| `default` | Standard HA cluster | Flannel | None (bare) |
| `calico` | Calico CNI testing | Calico | None (bare) |
| `cilium` | Cilium CNI testing | Cilium | None (bare) |
| `ipv6` | IPv6 dual-stack | Flannel | None (bare) |
| `kube-vip` | kube-vip LB testing | Flannel | None (bare) |
| `single_node` | Single-node cluster | Flannel | None (bare) |

### Note on Inventory Group Names

Molecule tests use `master` and `node` group names for backwards compatibility with upstream testing conventions. The sample inventory (`inventory/sample/hosts.yml.alternative`) uses modern `k3s_servers` and `k3s_workers` naming. Both conventions are valid and supported by the playbooks.

### Running Tests

**Run all scenarios:**
```bash
for scenario in default calico cilium ipv6 kube-vip single_node; do
  molecule test -s $scenario
done
```

**Run specific scenario with debugging:**
```bash
ANSIBLE_VERBOSITY=2 molecule test -s default
```

**Run without destroying (for debugging):**
```bash
molecule converge -s default  # Deploy
molecule verify -s default    # Verify
# ... investigate ...
molecule destroy -s default   # Cleanup when done
```

## Creating New Molecule Scenarios

### Scenario 1: Infrastructure Testing

Create `molecule/infrastructure/` to test deployment with infrastructure components.

**molecule/infrastructure/molecule.yml:**
```yaml
---
dependency:
  name: galaxy
driver:
  name: vagrant
platforms:
  - name: infra-control1
    box: generic/ubuntu2204
    memory: 2048  # More memory for infrastructure
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
    interfaces:
      - network_name: private_network
        ip: 192.168.31.10

  - name: infra-control2
    box: generic/ubuntu2204
    memory: 2048
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
    interfaces:
      - network_name: private_network
        ip: 192.168.31.11

  - name: infra-control3
    box: generic/ubuntu2204
    memory: 2048
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
    interfaces:
      - network_name: private_network
        ip: 192.168.31.12

  - name: infra-worker1
    box: generic/ubuntu2204
    memory: 2048
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_workers
    interfaces:
      - network_name: private_network
        ip: 192.168.31.20

provisioner:
  name: ansible
  env:
    ANSIBLE_VERBOSITY: 1
  playbooks:
    converge: ../resources/converge.yml
    verify: ../resources/verify_infrastructure.yml
  inventory:
    links:
      group_vars: ../../inventory/sample/group_vars

scenario:
  test_sequence:
    - dependency
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - verify
    - cleanup
    - destroy
```

**molecule/infrastructure/overrides.yml:**
```yaml
---
- name: Apply overrides
  hosts: all
  tasks:
    - name: Override host variables
      ansible.builtin.set_fact:
        flannel_iface: eth1
        retry_count: 45

        # Cluster provisioning only
        apiserver_endpoint: 192.168.31.100
        metal_lb_ip_range: 192.168.31.200-192.168.31.210
```

## Infrastructure Testing - DEPRECATED

Infrastructure testing has been moved to k8s-homelab.

k3s-ansible testing now focuses on:

### Cluster Provisioning
- [ ] k3s installation completes successfully
- [ ] All nodes join cluster
- [ ] Control plane is accessible
- [ ] kube-vip VIP is stable
- [ ] MetalLB assigns LoadBalancer IPs

### CNI Testing
- [ ] Pods can communicate across nodes
- [ ] DNS resolution works (CoreDNS)
- [ ] Service endpoints are reachable

### Storage Testing
- [ ] k3s local-storage provisioner creates PVCs (if enabled)
- [ ] PVs are bound to PVCs

For infrastructure testing (ingress, Longhorn, Prometheus), see k8s-homelab.
```

### Scenario 2: Join Cluster Testing

Create `molecule/join/` to test the join_cluster.yml playbook.

**molecule/join/molecule.yml:**
```yaml
---
dependency:
  name: galaxy
driver:
  name: vagrant
platforms:
  # Initial 3-server cluster
  - name: join-control1
    box: generic/ubuntu2204
    memory: 1024
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
      - initial_cluster
    interfaces:
      - network_name: private_network
        ip: 192.168.32.10

  - name: join-control2
    box: generic/ubuntu2204
    memory: 1024
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
      - initial_cluster
    interfaces:
      - network_name: private_network
        ip: 192.168.32.11

  - name: join-control3
    box: generic/ubuntu2204
    memory: 1024
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
      - initial_cluster
    interfaces:
      - network_name: private_network
        ip: 192.168.32.12

  # New nodes to join
  - name: join-control4
    box: generic/ubuntu2204
    memory: 1024
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_servers
      - new_nodes
    interfaces:
      - network_name: private_network
        ip: 192.168.32.13

  - name: join-worker1
    box: generic/ubuntu2204
    memory: 1024
    cpus: 2
    groups:
      - k3s_cluster
      - k3s_workers
      - new_nodes
    interfaces:
      - network_name: private_network
        ip: 192.168.32.20

provisioner:
  name: ansible
  env:
    ANSIBLE_VERBOSITY: 1
  playbooks:
    converge: converge_join.yml
    verify: verify_join.yml
  inventory:
    links:
      group_vars: ../../inventory/sample/group_vars

scenario:
  test_sequence:
    - dependency
    - cleanup
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - verify
    - cleanup
    - destroy
```

**molecule/join/overrides.yml:**
```yaml
---
- name: Apply overrides
  hosts: all
  tasks:
    - name: Override host variables
      ansible.builtin.set_fact:
        flannel_iface: eth1
        retry_count: 45
        apiserver_endpoint: 192.168.32.100
        metal_lb_ip_range: 192.168.32.200-192.168.32.210
        deploy_infrastructure: false
```

**molecule/join/converge_join.yml:**
```yaml
---
# Step 1: Deploy initial cluster (first 3 servers)
- name: Apply overrides
  ansible.builtin.import_playbook: ../resources/overrides_loader.yml

- name: Deploy initial cluster
  ansible.builtin.import_playbook: ../../site.yml
  when: inventory_hostname in groups['initial_cluster']

# Step 2: Get cluster token
- name: Get cluster token from existing cluster
  hosts: join-control1
  tasks:
    - name: Read node token
      ansible.builtin.slurp:
        src: /var/lib/rancher/k3s/server/node-token
      register: node_token_file

    - name: Set cluster token fact
      ansible.builtin.set_fact:
        existing_cluster_token: "{{ node_token_file.content | b64decode | trim }}"

    - name: Share token with all hosts
      ansible.builtin.add_host:
        name: "{{ item }}"
        existing_cluster_token: "{{ existing_cluster_token }}"
      loop: "{{ groups['new_nodes'] }}"

# Step 3: Join new nodes
- name: Set join variables for new nodes
  hosts: new_nodes
  tasks:
    - name: Configure join parameters
      ansible.builtin.set_fact:
        join_existing_cluster: true
        existing_cluster_apiserver: "{{ hostvars['join-control1']['apiserver_endpoint'] }}"
        existing_cluster_token: "{{ hostvars[inventory_hostname]['existing_cluster_token'] }}"

- name: Join new nodes to cluster
  ansible.builtin.import_playbook: ../../join_cluster.yml
  vars:
    ansible_limit: new_nodes
```

**molecule/join/verify_join.yml:**
```yaml
---
- name: Verify join cluster
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Get kubeconfig
      ansible.builtin.set_fact:
        kubeconfig: "{{ lookup('env', 'HOME') }}/.kube/k3s-ansible-test-config"

    - name: Wait for all nodes to be ready
      ansible.builtin.command:
        cmd: kubectl --kubeconfig {{ kubeconfig }} get nodes
      register: nodes
      until: nodes.stdout_lines | select('search', 'Ready') | list | length == 5
      retries: 30
      delay: 10
      changed_when: false

    - name: Verify 4 servers in etcd
      ansible.builtin.command:
        cmd: kubectl --kubeconfig {{ kubeconfig }} -n kube-system exec -it deploy/coredns -- sh -c "etcdctl member list"
      register: etcd_members
      failed_when: etcd_members.stdout_lines | length != 4
      changed_when: false
      ignore_errors: true

    - name: Verify new worker node joined
      ansible.builtin.command:
        cmd: kubectl --kubeconfig {{ kubeconfig }} get node join-worker1
      changed_when: false

    - name: Display cluster status
      ansible.builtin.debug:
        msg:
          - "Join cluster test successful!"
          - "Total nodes: {{ nodes.stdout_lines | length }}"
          - "Servers: 4 (initial 3 + joined 1)"
          - "Workers: 1 (joined)"
```

## Manual Testing Checklist

For comprehensive manual testing, see [docs/testing-checklist.md](docs/testing-checklist.md).

### Quick Manual Test: Bare Cluster

```bash
# 1. Setup inventory
cp -R inventory/sample/group_vars inventory/group_vars
cp inventory/sample/hosts.yml inventory/hosts.yml

# Edit hosts.yml with your servers/workers

# 2. Configure (ensure infrastructure disabled)
# In inventory/group_vars/all/vars.yml:
deploy_infrastructure: false  # or omit

# 3. Deploy
ansible-playbook site.yml -i inventory/hosts.yml

# 4. Verify
kubectl get nodes
kubectl get pods -A

# Expected: k3s, kube-vip, MetalLB, CoreDNS only
# No ingress, longhorn, prometheus, etc.
```

### Infrastructure Deployment

Infrastructure components are no longer included in k3s-ansible.

For infrastructure deployment (ingress, storage, monitoring), see [k8s-homelab](https://github.com/kriegalex/k8s-homelab).

### Quick Manual Test: Join Existing Cluster

```bash
# 1. Get token from existing cluster
ssh user@existing-server
sudo cat /var/lib/rancher/k3s/server/node-token
# Copy output

# 2. Configure
# In inventory/group_vars/all/vars.yml:
join_existing_cluster: true
existing_cluster_apiserver: "192.168.1.100"  # VIP of existing cluster

# In inventory/group_vars/all/vault.yml:
existing_cluster_token: "K1234567890abcdef::server:abcdef1234567890"

# 3. Add new node to inventory (hosts.yml)
# 4. Deploy (limit to new node only!)
ansible-playbook join_cluster.yml -i inventory/hosts.yml --limit new_server_hostname

# 5. Verify
kubectl get nodes  # Should show new node
```

## CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/test.yml
name: Test k3s-ansible

on:
  push:
    branches: [main, develop]
  pull_request:

jobs:
  molecule:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        scenario:
          - default
          - calico
          - cilium
          - single_node
    steps:
      - uses: actions/checkout@v3

      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install molecule ansible-lint
          ansible-galaxy collection install -r collections/requirements.yml

      - name: Run Molecule test
        run: molecule test -s ${{ matrix.scenario }}
        env:
          ANSIBLE_FORCE_COLOR: '1'
```

## Troubleshooting

### Molecule tests fail with "Connection refused"

**Issue:** VMs not accessible via Vagrant

**Solution:**
```bash
# Check Vagrant VMs
vagrant global-status

# Destroy stuck VMs
molecule destroy -s default

# Try again
molecule test -s default
```

### Tests timeout waiting for nodes

**Issue:** Nodes take too long to join

**Solution:**
- Increase `retry_count` in overrides.yml
- Allocate more CPU/RAM to VMs in molecule.yml
- Check VM performance: `vagrant ssh <vm-name>`

### Infrastructure tests fail

**Issue:** Infrastructure components don't deploy

**Solution:**
- Ensure VMs have sufficient resources (2GB+ RAM)
- Check Helm is installed: `ansible-playbook` logs
- Verify network connectivity between VMs

## Next Steps

1. ✅ Run existing Molecule scenarios to verify bare cluster deployment
2. ⚠️ Create infrastructure scenario (optional - template provided above)
3. ⚠️ Create join scenario (optional - template provided above)
4. ✅ Run manual testing checklist (see docs/testing-checklist.md)
5. ✅ Test on real hardware before release

## Testing Completion Criteria

Before releasing refactored version:

- [ ] All 6 existing Molecule scenarios pass
- [ ] Manual bare cluster deployment successful
- [ ] Manual infrastructure deployment successful
- [ ] Manual join cluster successful
- [ ] Documentation tested and accurate
- [ ] No regressions in core functionality
