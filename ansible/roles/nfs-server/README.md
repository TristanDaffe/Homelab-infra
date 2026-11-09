# nfs-server

Installs the Linux NFS server and publishes the directories declared in
`nfs_server_exports`.

## Requirements

- Ansible ≥ 2.15
- Debian 12+ / Ubuntu 22.04+

## Role variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `nfs_server_exports` | `[]` | Export definitions; an empty list leaves the role disabled by the consumer |
| `nfs_server_package` | `nfs-kernel-server` | Package providing the NFS server |
| `nfs_server_service` | `nfs-server` | Service name |
| `nfs_server_exports_file` | `/etc/exports.d/ansible.exports` | Managed exports file |
| `nfs_server_firewall_enabled` | `true` | Add UFW rules for NFS traffic |
| `nfs_server_firewall_sources` | `[]` | Networks/hosts allowed to access NFS |

Each export has a local `path`, optional ownership/mode, and a `clients` list.
Each client has a `network` or host address and optional NFS `options`.

```yaml
nfs_server_exports:
  - path: /srv/nfs
    clients:
      - network: 192.168.129.0/23
        options: rw,sync,no_subtree_check
nfs_server_firewall_sources:
  - 192.168.129.0/23
```

The role creates the exported directories and reloads exports when their
configuration changes. Client mounts and Kubernetes storage classes are
consumer responsibilities.

## Example

```yaml
- hosts: storage
  roles:
    - role: nfs-server
      when: nfs_server_exports | length > 0
```

## Dependencies

None.
