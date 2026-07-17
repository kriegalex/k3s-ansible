# k3s cluster provisioning (official k3s-io/k3s-ansible + homelab config)

This repo provisions the homelab k3s cluster using the **official
[k3s-io/k3s-ansible](https://github.com/k3s-io/k3s-ansible)** playbook,
consumed read-only as a pinned git submodule (`k3s-io-ansible/`). Everything
homelab-specific is expressed as inventory variables plus one static manifest —
**no upstream file is ever modified**, so updating upstream can never produce a
merge conflict.

The previous techno-tim–based playbook is preserved on the
[`archive/timothy-fork`](../../tree/archive/timothy-fork) branch.

## Layout

| Path | Purpose |
|---|---|
| `site.yml` | Thin wrapper: `import_playbook` of the upstream site playbook |
| `k3s-io-ansible/` | Pinned upstream submodule (never edited) |
| `inventory.yml` | Hosts; groups `server` / `agent` / `k3s_cluster` (names required by upstream) |
| `group_vars/all/vars.yml` | k3s version, api endpoint, kubeconfig context |
| `group_vars/all/vault.yml` | `token:` — ansible-vault encrypted, **not** committed (see below) |
| `group_vars/server.yml` | `server_config_yaml` (all server flags) + `extra_manifests` (MetalLB) |
| `group_vars/agent.yml` | `agent_config_yaml` |
| `manifests/metallb-crds.yaml` | Pinned MetalLB v0.14.8 + `first-pool` (10.0.0.20-49) + L2Advertisement |

Design notes:

- All k3s flags live in `/etc/rancher/k3s/config.yaml` (written by the upstream
  role from `server_config_yaml` / `agent_config_yaml`). `token` and
  `tls-san: {{ api_endpoint }}` are auto-injected by the role — never duplicate
  them in the config vars.
- MetalLB is deployed via upstream's `extra_manifests` mechanism: the file is
  copied to `/var/lib/rancher/k3s/server/manifests/` and k3s's AddOn controller
  applies it. The filename `metallb-crds.yaml` deliberately matches the AddOn
  created by the old playbook, so the running MetalLB is adopted in place.
- kube-vip was removed (single control-plane node; the "VIP" was the server's
  own IP).

## Usage

Always run from the repo root (`ansible.cfg` is cwd-loaded):

```bash
git submodule update --init                        # first checkout only
ansible-galaxy collection install -r collections/requirements.yml
ansible-playbook site.yml                          # full converge
ansible-playbook site.yml --check --diff           # drift check (see baseline below)
ansible-playbook site.yml --limit agent --forks 1  # agents one at a time
```

**Adding a node**: add it to `inventory.yml` under `agent`, then
`ansible-playbook site.yml --limit <new-node>`. (The old `join_cluster.yml` is
obsolete — the upstream agent role joins idempotently via the shared token.)

**Upgrading k3s**: bump `k3s_version` in `group_vars/all/vars.yml`, then run
`site.yml` (single server; on multi-server use `--forks=1`), or use
`k3s-io-ansible/playbooks/upgrade.yml`.

**Updating upstream**: `cd k3s-io-ansible && git fetch && git checkout <tag>`,
review upstream changelog, commit the new submodule pointer. Conflict-free by
construction.

## Vault (required before any real run)

The cluster token must match the live cluster byte-for-byte so nodes keep their
identity:

```bash
# on k3s-server1:
sudo cat /var/lib/rancher/k3s/server/token
# locally:
echo '<vault-password>' > .vault_pass && chmod 600 .vault_pass
ansible-vault create group_vars/all/vault.yml     # token: "<paste>"
```

Then uncomment `vault_password_file = .vault_pass` in `ansible.cfg`.

See `group_vars/all/vault.yml.example`. **Warning:** if `token` is undefined,
the upstream role generates a random one — on an existing cluster that would
break agent auth. Do not run `site.yml` against the cluster without the vault.

## `--check --diff` baseline (expected residual noise)

On a fully converged cluster, `ansible-playbook site.yml --check --diff`
reports **exactly 4 changed** tasks — the upstream roles restart services
unconditionally by design (declarative upgrade path):

1. `Enable and start K3s service` (k3s-server1, `state: restarted`)
2. `Enable and start K3s agent` × 3 (workers)

**Anything beyond these 4 is real drift — investigate.** Note that the
config.yaml writes are check-mode-gated upstream, so `--check` cannot reveal
config drift; verify `/etc/rancher/k3s/config.yaml` on the nodes directly.

Known quirk: **before the first real converge**, `--check` fails on the server
at `Setup kubeconfig context on control node` (`~/.kube/config.new` doesn't
exist yet — the fetch that creates it is check-gated upstream). Harmless;
disappears after the first real run, when `Copy k3s.yaml to second file`
reports `ok` and the whole kubeconfig block is skipped.

## Migration runbook (techno-tim → official, one-time)

Server first, then agents. Rehearse on throwaway VMs if possible
(`archive/timothy-fork` has the molecule/Vagrant tooling to build a
Timothy-style victim cluster).

1. **Pre-flight**
   - Create the vault (above).
   - Passwordless sudo for `{{ ansible_user }}` on **all** nodes:
     `ansible k3s_cluster -m command -a whoami --become` must succeed everywhere.
     (k3s-worker4 was missing this as of 2026-07-17 — on the node:
     `echo 'k3s ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/k3s && sudo chmod 440 /etc/sudoers.d/k3s`)
   - Confirm `/usr/local/bin/k3s` exists on all nodes (install script then
     skips the download; version is unchanged).
   - Backup units for rollback — server:
     `sudo cp /etc/systemd/system/k3s.service{,.bak}`; each worker:
     `sudo cp /etc/systemd/system/k3s-node.service{,.bak}`.
     (Unit naming: the old playbook used `k3s.service` on the server — same
     name as the official one, overwritten in place — but `k3s-node.service`
     on workers, while the official install creates `k3s-agent.service`.
     The old worker unit must therefore be retired during cutover, step 5.)
   - Record LB services: `kubectl get svc -A | grep LoadBalancer`
     (expect 10.0.0.20–.27).
   - `ansible-inventory --graph` and `ansible-playbook site.yml --syntax-check`.
2. **kube-vip cleanup** (files BEFORE objects, or the AddOn controller
   recreates them) — on k3s-server1:
   ```bash
   sudo rm -f /var/lib/rancher/k3s/server/manifests/vip.yaml \
              /var/lib/rancher/k3s/server/manifests/vip-rbac.yaml
   kubectl -n kube-system delete addon vip vip-rbac
   kubectl -n kube-system delete daemonset kube-vip-ds
   ```
3. **Server cutover**: `ansible-playbook site.yml --limit server`
   — rewrites the systemd unit to the official install-script form, writes
   config.yaml, adopts the MetalLB AddOn, restarts k3s once (~10–30 s API
   blip; MetalLB data plane and workloads unaffected; etcd data untouched).
4. **Verify**: nodes Ready, LB IPs unchanged, `kubectl -n metallb-system get
   pods`, `kubectl get ipaddresspool -n metallb-system first-pool`.
5. **Agent cutover** — one worker at a time; the old `k3s-node.service` must
   be stopped *before* the new `k3s-agent.service` starts, or two agents race
   for the kubelet port. Per worker:
   ```bash
   # on the worker:
   sudo systemctl disable --now k3s-node.service   # containers keep running (containerd survives)
   # from the control node:
   ansible-playbook site.yml --limit <worker>
   kubectl get nodes                               # wait for Ready
   # once verified, on the worker (the .bak from pre-flight remains):
   sudo rm /etc/systemd/system/k3s-node.service && sudo systemctl daemon-reload
   ```
6. **Post**: run the `--check --diff` baseline; diff each node's
   `/etc/rancher/k3s/config.yaml` and the `k3s.io/node-args` annotation
   against pre-migration values.

**Rollback**: server — restore `k3s.service` from `.bak`, `sudo systemctl
daemon-reload && sudo systemctl restart k3s`. Worker — `sudo systemctl
disable --now k3s-agent.service`, restore `k3s-node.service` from `.bak`,
`sudo systemctl daemon-reload && sudo systemctl enable --now k3s-node`.
Data and binary are untouched either way.

Once the new setup has survived a re-run and a reboot cycle, the legacy
untracked `inventory/` directory can be deleted.
