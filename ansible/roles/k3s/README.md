# k3s

Installs a [k3s](https://k3s.io) cluster on provisioned nodes using the official
`get.k3s.io` install script.

- Every host in **`k3s_server`** is a server (control plane + workloads) with
  **embedded etcd**. The first one bootstraps the cluster (`--cluster-init`) and
  exposes its join token as a fact; the others join it as servers in the same
  play. Use an odd number (3): etcd needs a majority, so 2 servers tolerate no
  failure.
- Hosts in the optional **`k3s_agent`** group join as workers.

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
- A joining server can't be installed alone (`--limit`): it needs the first
  server's token, set earlier in the same play.
- Where UFW is installed (e.g. the `security` role ran on the node), the role
  opens what k3s needs: the pod/service networks, 8472/udp (flannel VXLAN) and
  10250/tcp (kubelet) between nodes, 2379-2380/tcp (etcd) between servers,
  6443/tcp (API) on servers from
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
- Longhorn prerequisites (`--tags k3s-longhorn`): `open-iscsi` + `nfs-common`,
  `iscsid` running, `iscsi_tcp` loaded, multipathd blacklisting `sd*` (it would
  grab Longhorn's iSCSI disks), and the data disk `k3s_longhorn_disk` (scsi1 from
  proxmox-vm `data_disk_gb`) formatted ext4 once and mounted at `/var/lib/longhorn`.
- `k3s_kube_apiserver_args` (config.yaml `kube-apiserver-arg`, servers): default
  30 s instead of 300 s before pods leave an unreachable/not-ready node. Applies
  to pods created afterwards. Config changes restart the servers one at a time
  (`--tags k3s-config`), each waiting for its API before the next.
