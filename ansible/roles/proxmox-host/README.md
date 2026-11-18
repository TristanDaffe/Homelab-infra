# proxmox-host

Configures a Proxmox VE host: powertop tuning, snippet/template directories, cloud-init `user-data` snippet, Ubuntu cloud image + selected LXC template downloads, CIFS mounts, and iGPU subgid mapping for unprivileged LXC passthrough.

## Requirements

- Ansible ≥ 2.15
- `ansible.posix` collection (CIFS mounts)
- Target host is a Proxmox VE node (Debian-based)

## Role variables

| Variable                     | Default                                                                       | Purpose                                            |
| ---------------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `ubuntu_lts_image_url`       | https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img | Ubuntu LTS cloud image URL                         |
| `ubuntu_cloudimg_filename`   | basename of the URL                                                            | Image filename (also used by `proxmox-vm`)         |
| `ubuntu_lxc_template`        | `ubuntu-24.04-standard_24.04-2_amd64.tar.zst`                                  | LXC template (must exist in `pveam available`)     |
| `debian_lxc_template`   | `debian-13-standard_13.6-1_amd64.tar.zst`                                      | Debian 13 template for containers selecting it      |
| `cloud_init_keyboard_layout` | `be`                                                                           | XKB layout for the cloud-init snippet              |
| `cifs_mounts`                | `[]`                                                                           | List of `{ src, path, opts }` host-level mounts    |
| `proxmox_host_nic_offload_off` | `[]`                                                                        | Interfaces with TSO/GSO turned off (e1000e hangs)  |

External: the `igpu` tasks read `lxc_containers` (nodesetup group_vars) to detect unprivileged containers with `enable_igpu: true`.

## Facts exported

`igpu_video_gid` and `igpu_render_gid` (host `video`/`render` group GIDs) — consumed by the **proxmox-lxc** role's idmap block. Running `--tags proxmox-lxc` alone skips these; include the `igpu` tag when provisioning iGPU containers.

## Tags

- `setup`  — host setup only (powertop, dirs, downloads)
- `cifs`   — CIFS mounts only
- `igpu`   — iGPU subgid mapping only
- `nic`    — NIC offload settings only. Sets a `post-up ethtool -K <nic> tso off gso off` in
  `/etc/network/interfaces` and applies it now (the link resets for a few seconds).
  The Intel I219 (`e1000e`) of the Lenovo Tiny hosts hangs under sustained gigabit
  traffic otherwise. Set on pve-dev and pve-dev-2; prod `pve` showed the same hangs
  in 2025 but none since 2026-03 on its 10/100 link: add it there if it moves to gigabit.

## Example

```yaml
- hosts: proxmox
  roles:
    - role: proxmox-host
```

## Dependencies

None.

## License

MIT
