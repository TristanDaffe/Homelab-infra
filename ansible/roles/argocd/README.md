# argocd

Bootstraps [Argo CD](https://argo-cd.readthedocs.io/) on the k3s cluster and
points it at `Susu/Homelab-k8S`. Runs on the first host of `k3s_server`.

1. Installs Helm (`argocd_helm_version`, checksum verified) and `python3-kubernetes`.
2. Installs the `argo-cd` chart **only if the `argocd` namespace does not exist**,
   with the minimal `argocd_bootstrap_values`. From then on Argo CD manages
   itself from `Homelab-k8S/bootstrap/argocd.yaml` — upgrade it there.
3. Creates the repository Secret (SSH, read-only deploy key `argocd_repo_ssh_key`
   from SOPS) and the `root` Application syncing `bootstrap/`.

```bash
cd ansible
ansible-playbook -i configure/inventory-dev configure/argocd.yaml
```

The public half of the deploy key must be added to Homelab-k8S → Settings →
Deploy keys (read-only), otherwise `root` shows "repository not accessible".

| Variable | Default | Purpose |
|---|---|---|
| `argocd_helm_version` | `v4.3.1` | Helm CLI on the node |
| `argocd_chart_version` | `10.17.1` | Chart for the first install only |
| `argocd_repo_url` | `ssh://git@git.autostack.ovh:222/Susu/Homelab-k8S.git` | Repo synced by `root` |
| `argocd_root_path` | `bootstrap` | Folder the `root` Application syncs |
| `argocd_bootstrap_values` | minimal | Values for the first install (repo SSH host key, dex/notifications off) |
| `argocd_repo_ssh_key` | — (SOPS) | Private deploy key |
