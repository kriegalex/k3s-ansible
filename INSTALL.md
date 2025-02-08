# Development Environment Setup Guide

This guide explains how to set up the required tools for Kubernetes and Ansible development.

## Prerequisites

The following packages will be installed on Ubuntu 24.04 LTS:
- python3-pip
- python3-venv
- pipx
- ansible-lint
- kubectl
- helm

## Installation Steps

### 1. System Packages

```bash
# Update package list
sudo apt update

# Install Python dependencies
sudo apt install -y python3-pip python3-venv pipx
```

### 2. Install Tools via pipx

```bash
# Ensure pipx is in PATH
pipx ensurepath

# Install ansible-lint
pipx install --include-deps ansible

# Install kubernetes python client
pipx inject --include-deps ansible ansible-lint kubernetes netaddr
```

### 3. Install Helm

```bash
# Add Helm repository
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# Update and install Helm
sudo apt update
sudo apt install -y helm
helm plugin install https://github.com/databus23/helm-diff
```

## Configuration
### Visual Studio Code Settings
Add the following to your VS Code settings.json to use the pipx Python environment:
```json
{
    "python.defaultInterpreterPath": "${HOME}/.local/share/pipx/venvs/ansible/bin/python",
    "python.analysis.extraPaths": [
        "${HOME}/.local/share/pipx/venvs/ansible/lib/python3.11/site-packages"
    ]
}
```

### Ansible Configuration
Create or modify your ansible.cfg file to include the Python interpreter path:

```ini
[defaults]
interpreter_python = ${HOME}/.local/share/pipx/venvs/ansible/bin/python
```

## Verification
Verify the installations with the following commands:

```bash
# Check ansible-lint version
ansible-lint --version

# Check kubectl version
kubectl version --client

# Check Helm version
helm version
```

## Path Information
Default installation paths:

pipx environments: ${HOME}/.local/share/pipx/venvs/
Binary locations: ${HOME}/.local/bin/

## Troubleshooting
If you encounter PATH-related issues, ensure your ~/.bashrc or ~/.zshrc includes:

```
export PATH="$HOME/.local/bin:$PATH"
```

You may need to reload your shell or run:

```bash
source ~/.bashrc  # or source ~/.zshrc for Zsh users
```

Note: Replace python3.11 in paths with your actual Python version if different.