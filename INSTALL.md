# Development Environment Setup Guide

This guide explains how to set up the required tools for Kubernetes and Ansible development.

## Quick Start

Choose your Linux distribution:
- **[Ubuntu/Debian](#ubuntudebian-2404-lts)** - Tested and supported
- **[Fedora/RHEL/Rocky](#fedorarhel)** - Tested on Rocky 9
- **[Arch Linux](#arch-linux)** - Community supported
- **[Other Distributions](#other-distributions)** - Universal binary method

All distributions require:
- Python 3.9+
- Ansible 2.11+
- kubectl (for cluster management)
- helm (for application deployment)

---

## Ubuntu/Debian (24.04 LTS)

### Why pipx?
Ubuntu 23.04+ implements PEP 668, preventing direct `pip install` into system Python.
We use pipx to isolate Ansible in its own virtual environment.

### Installation

#### 1. System Packages

```bash
# Update package list
sudo apt update

# Install Python dependencies
sudo apt install -y python3-pip python3-venv pipx
```

#### 2. Install Ansible via pipx

```bash
# Ensure pipx is in PATH
pipx ensurepath

# Install ansible with dependencies
pipx install --include-deps ansible

# Install additional Python packages needed for k3s-ansible
pipx inject --include-deps ansible ansible-lint kubernetes netaddr jmespath
```

#### 3. Install kubectl

**Option A: Binary Download (Recommended for currency)**

```bash
# Download latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Verify binary (optional but recommended)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Option B: Package Manager**

```bash
# Install from apt (may be older version)
sudo apt-get update
sudo apt-get install -y kubectl
```

#### 4. Install Helm

**Option A: Native Package (Recommended)**

```bash
# Add Helm repository
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# Update and install Helm
sudo apt update
sudo apt install -y helm
helm plugin install https://github.com/databus23/helm-diff
```

**Option B: Binary Download**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm plugin install https://github.com/databus23/helm-diff
```

---

## Fedora/RHEL

Fedora and RHEL-based distributions (Rocky, AlmaLinux, CentOS Stream) can install
dependencies directly via dnf without pipx isolation.

### Installation

#### 1. System Packages & Ansible

**Fedora:**

```bash
sudo dnf install -y ansible ansible-lint python3-pip python3-kubernetes python3-netaddr python3-jmespath
```

**RHEL/Rocky/AlmaLinux:**

```bash
# Enable EPEL repository first
sudo dnf install -y epel-release

# Install Ansible
sudo dnf install -y ansible python3-pip

# Install additional dependencies via pip
pip3 install ansible-lint kubernetes netaddr jmespath
```

#### 2. Install kubectl

**Option A: Binary Download (Recommended for currency)**

```bash
# Download latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Verify binary (optional but recommended)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Option B: Package Manager**

```bash
# Install from dnf (may be older version)
sudo dnf install -y kubectl
```

#### 3. Install Helm

```bash
# Using dnf
sudo dnf install -y helm

# Configure helm-diff plugin
helm plugin install https://github.com/databus23/helm-diff
```

### Configuration
Ansible is installed system-wide, no special Python path configuration needed.

---

## Arch Linux

Arch Linux has all dependencies available in official repositories or AUR,
with no PEP 668 restrictions. No pipx needed!

### Installation

#### 1. Core Packages

```bash
# Install from official repos
sudo pacman -S ansible ansible-lint python-kubernetes python-netaddr python-jmespath helm
```

#### 2. Install kubectl

**Option A: Binary Download (Recommended for currency)**

```bash
# Download latest kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Verify binary (optional but recommended)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

# Install
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

**Option B: AUR Package**

```bash
# Using paru (or yay)
paru -S kubectl-bin
```

#### 3. Helm Plugin

```bash
helm plugin install https://github.com/databus23/helm-diff
```

### Configuration
All packages are system-wide, no special configuration required.

---

## Other Distributions

For distributions not listed above, use the universal binary installation method:

### Universal Binary Method

#### 1. Python & Ansible

```bash
# Install Python 3.9+ via your package manager
# Then install Ansible via pip:
python3 -m pip install --user ansible ansible-lint kubernetes netaddr jmespath

# Add to PATH if needed
export PATH="$HOME/.local/bin:$PATH"
```

#### 2. kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

#### 3. Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm plugin install https://github.com/databus23/helm-diff
```

---

## Configuration (All Distributions)

### Visual Studio Code Settings

**For Ubuntu/Debian users (using pipx):**

```json
{
    "python.defaultInterpreterPath": "${HOME}/.local/share/pipx/venvs/ansible/bin/python",
    "python.analysis.extraPaths": [
        "${HOME}/.local/share/pipx/venvs/ansible/lib/python3.11/site-packages"
    ]
}
```

**For Fedora/RHEL/Arch users (system-wide install):**

```json
{
    "python.defaultInterpreterPath": "/usr/bin/python3"
}
```

### Ansible Configuration

**Ubuntu/Debian (pipx):**

Create or modify your ansible.cfg file to include the Python interpreter path:

```ini
[defaults]
interpreter_python = ${HOME}/.local/share/pipx/venvs/ansible/bin/python
```

**Other distributions:**

```ini
[defaults]
interpreter_python = /usr/bin/python3
```

---

## Verification (All Distributions)

Verify the installations with the following commands:

```bash
# Check ansible-lint version
ansible-lint --version

# Check kubectl version
kubectl version --client

# Check Helm version
helm version
```

---

## Troubleshooting

### PATH Issues

**Ubuntu/Debian (pipx):**

```bash
# Ensure pipx binaries are in PATH
export PATH="$HOME/.local/bin:$PATH"
```

**All distributions:**

```bash
# Ensure kubectl and helm are accessible
which kubectl helm ansible

# If missing, add to ~/.bashrc or ~/.zshrc:
export PATH="/usr/local/bin:$PATH"
```

You may need to reload your shell or run:

```bash
source ~/.bashrc  # or source ~/.zshrc for Zsh users
```

### Python Version Conflicts

**Arch Linux:**

Arch uses Python 3.12+. If you encounter compatibility issues:

```bash
# Check Python version
python --version

# Ensure Ansible uses correct Python
ansible --version | grep "python version"
```

### Ansible Collection Installation

If you encounter issues installing collections:

```bash
# Clear collection cache
rm -rf ~/.ansible/collections

# Reinstall with specific Python
python3 -m pip install --upgrade ansible
ansible-galaxy collection install -r ./collections/requirements.yml
```

### Permission Errors (PEP 668 on Ubuntu)

If you see "externally-managed-environment" error on Ubuntu:
- ✅ Solution: Use pipx (recommended method above)
- ⚠️ Workaround: Use virtual environments instead of system pip
- ❌ Don't: Use `pip install --break-system-packages` (dangerous)

### Path Information

**Ubuntu/Debian (pipx):**
- pipx environments: `${HOME}/.local/share/pipx/venvs/`
- Binary locations: `${HOME}/.local/bin/`

**Other distributions:**
- System Python packages: `/usr/lib/python3.X/site-packages/`
- User Python packages: `${HOME}/.local/lib/python3.X/site-packages/`
- Binary locations: `/usr/bin/`, `/usr/local/bin/`

---

## Deployment Modes Quick Reference

After setting up your development environment, you can deploy k3s clusters in three modes:

### 1. Bare Cluster (Default)
Minimal k3s cluster with no additional infrastructure.

```bash
# Configure
# inventory/group_vars/all/vars.yml:
# deploy_infrastructure: false  # or omit

# Deploy
ansible-playbook site.yml -i inventory/hosts.yml
```

**What you get:** k3s, kube-vip, MetalLB, CoreDNS
**No infrastructure:** ingress, storage, database, monitoring

### 2. Cluster + Infrastructure
k3s cluster with optional infrastructure components.

```bash
# Deploy k3s cluster only
ansible-playbook site.yml -i inventory/hosts.yml

# For infrastructure deployment, see k8s-homelab repository:
# https://github.com/kriegalex/k8s-homelab
```

**What you get:** Bare k3s cluster ready for infrastructure deployment

### 3. Join Existing Cluster
Add nodes to an existing k3s cluster.

```bash
# Configure
# inventory/group_vars/all/vars.yml:
join_existing_cluster: true
existing_cluster_apiserver: "192.168.1.100"

# inventory/group_vars/all/vault.yml:
existing_cluster_token: "K1234567890abcdef::server:abcdef1234567890"

# Deploy (limit to new nodes only)
ansible-playbook join_cluster.yml -i inventory/hosts.yml --limit new_server
```

**What you get:** New nodes joined to existing cluster without reinitialization

### Infrastructure & Application Management

**Infrastructure and applications are NOT managed by k3s-ansible.**

k3s-ansible focuses solely on cluster provisioning. For infrastructure and applications, use [k8s-homelab](https://github.com/kriegalex/k8s-homelab):

**Available in k8s-homelab:**
- **Ingress:** NGINX, Traefik, cert-manager
- **Storage:** Longhorn, NFS, Ceph
- **Database:** CloudNativePG, MySQL operators
- **Monitoring:** Prometheus, Grafana, Loki
- **Backup:** Velero, k8up
- **Applications:** Nextcloud, Plex, Gitea, Immich, and more

### Further Reading

- [README.md](README.md) - Complete deployment guide
- [TESTING.md](TESTING.md) - Testing guide with molecule scenarios
- [k8s-homelab](https://github.com/kriegalex/k8s-homelab) - Infrastructure and application management
