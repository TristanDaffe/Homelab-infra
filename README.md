# Homelab-infra

Infrastructure for the homelab: what runs outside Kubernetes, and what creates
and configures the machines. Kubernetes workloads live in `Homelab-k8S` (Argo CD).

| Path | Contents |
|---|---|
| `ansible/` | Ansible: Proxmox hosts (`nodesetup/`), node configuration and Docker Compose stacks (`configure/`), PostgreSQL HA, k3s install. Run every command from this directory. |
| `tofu/` | OpenTofu for the Proxmox VMs and LXCs (planned — will replace the VM/LXC part of `nodesetup/`). |
| `.forgejo/workflows/` | CI: lint on pull requests, apply `configure.yaml` on `main`. |

```bash
cd ansible
export SOPS_AGE_KEY_FILE=~/.config/sops/age/key.txt
ansible-galaxy collection install -r requirements.yml
ansible-playbook -i configure/inventory configure/configure.yaml
```

History before 2026-11-09 comes from `Susu/Ansible-homelab` (archived), rewritten
with `git filter-repo`: Swarm-era code removed, Ansible moved under `ansible/`.
