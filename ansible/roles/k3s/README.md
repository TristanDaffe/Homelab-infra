# k3s

Installs a [k3s](https://k3s.io) cluster on provisioned nodes using the official
`get.k3s.io` install script.

- Hosts in the **`k3s_server`** group become control-plane nodes. The first one
  is installed as the server, and its join token is read and exposed as a fact.
- Hosts in the **`k3s_agent`** group join that server as workers.

Run the two plays in order via `configure/kubernetes.yaml` (servers first, then
agents) so the agents can read the token from the server.

## Key variables

See `defaults/main.yaml`. Most-used:

| Variable | Default | Purpose |
|---|---|---|
| `k3s_version` | `""` (latest stable) | Pin, e.g. `v1.30.5+k3s1` |
| `k3s_server_extra_args` | `""` | e.g. `--disable traefik --disable servicelb` |
| `k3s_agent_extra_args` | `""` | Extra agent flags |
| `k3s_kubeconfig_mode` | `"600"` | Node kubeconfig perms (owner-only; holds admin creds) |
| `k3s_fetch_kubeconfig` | `false` | Copy kubeconfig to the controller (IP rewritten) |

## Usage

```bash
ansible-playbook -i configure/inventory-dev configure/kubernetes.yaml
# fetch a ready-to-use kubeconfig next to the playbook (configure/k3s-kubeconfig.yaml, gitignored):
ansible-playbook -i configure/inventory-dev configure/kubernetes.yaml -e k3s_fetch_kubeconfig=true
```

## Notes

- The install is guarded by `creates: /usr/local/bin/k3s`, so re-runs are no-ops
  once k3s is present — changing `*_extra_args` later won't re-apply on its own.
- Single control-plane (sqlite) by default. For embedded-etcd HA, add
  `--cluster-init` to the first server and `--server https://<first>:6443` to the
  others via `k3s_server_extra_args`.
- Where UFW is installed (e.g. the `security` role ran on the node), the role
  opens what k3s needs: the pod/service networks, 8472/udp (flannel VXLAN) and
  10250/tcp (kubelet) between nodes, 6443/tcp (API) on servers from
  `k3s_api_allowed_sources`. Apply only those rules with `--tags k3s-firewall`.
  Add 51820/udp yourself if you switch flannel to wireguard.
- Pods resolve through `k3s_pod_nameservers` (default the LAN Pi-hole), written
  to `/etc/rancher/k3s/resolv.conf` and set as `resolv-conf` in
  `/etc/rancher/k3s/config.yaml`. The node's own resolv.conf is Tailscale
  MagicDNS (`100.100.100.100`), which pods cannot reach. Apply only that part
  with `--tags k3s-config` (restarts k3s on changed nodes, then CoreDNS once).
- `k3s_disable` (default `[servicelb]`) is written as `disable:` in the servers'
  config.yaml: kube-vip (deployed by Argo CD from Homelab-k8S) owns the
  LoadBalancer IPs, and ServiceLB would fight it. Also applied by `--tags k3s-config`.
