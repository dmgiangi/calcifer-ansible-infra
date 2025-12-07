
This repository contains Ansible playbooks and roles to prepare an Ubuntu/Debian host and bootstrap a single‑control‑plane Kubernetes cluster with `kubeadm`. It installs prerequisites, configures containerd, installs Kubernetes packages, initializes the control plane, applies a CNI, and sets up a default StorageClass.

## Overview

- Stack: Ansible (YAML playbooks/roles)
- Target OS: Ubuntu/Debian-based hosts (APT, Ubuntu repo references present)
- Provisioning steps:
  - Sync hostname to match inventory
  - Configure kernel modules and sysctl for Kubernetes
  - Install and configure containerd (SystemdCgroup=true)
  - Install kubelet, kubeadm, kubectl from the Kubernetes apt repo
  - Initialize the control plane with `kubeadm init`
  - Apply CNI (Flannel by default)
  - Optionally install Local Path Provisioner and set it as default StorageClass

## Requirements

- Control machine:
  - Ansible installed (tested with modern 2.14+; exact minimum version TBD) — TODO: confirm minimal Ansible version
  - SSH access to target hosts and an SSH private key
  - Python 3 on control machine

- Managed hosts (targets):
  - Ubuntu/Debian-based system with `apt`
  - Python 3 available at `/usr/bin/python3` (see inventory)
  - User with sudo privileges (configured in inventory as `ansible_user`)
  - Swap will be disabled (runtime and `/etc/fstab` entries commented)

## Project Structure

```
.
├── ansible.cfg                 # Points to inventory and roles_path
├── inventory/
│   └── hosts.yml               # Inventory with groups: kubernetes_party, control_plane, workers
├── playbooks/
│   ├── init.yml                # Prepares hosts (hostname, prereqs, containerd, kube tools)
│   └── kubeadm_init.yml        # Initializes the control plane
└── roles/
    ├── containerd/             # Installs and configures containerd
    ├── hostname_sync/          # Ensures hostname matches inventory
    ├── k8s_prereqs/            # Kernel modules and sysctl, disables swap
    ├── kube_tools/             # Installs kubelet, kubeadm, kubectl
    └── kubeadm_init/           # Runs kubeadm init, applies CNI, storage
```

## Configuration

- ansible.cfg
  - `inventory = ./inventory/hosts.yml`
  - `roles_path = ./roles`

- Inventory (example included at `inventory/hosts.yml`):
  - Groups:
    - `kubernetes_party`: umbrella group for all Kubernetes nodes
    - `control_plane`: nodes where the control plane is initialized
    - `workers`: worker nodes (currently placeholder)
  - Host variables include:
    - `ansible_host`: SSH host/IP
    - `ansible_user`: SSH user
    - `ansible_ssh_private_key_file`: path to private key
    - `ansible_python_interpreter`: typically `/usr/bin/python3`

### Role defaults and important variables

- Role: `hostname_sync`
  - `hostname_target` (default: `{{ inventory_hostname }}`)

- Role: `k8s_prereqs`
  - No user‑facing variables; configures:
    - `/etc/modules-load.d/k8s.conf` with `overlay`, `br_netfilter`
    - sysctl: `bridge-nf-call-iptables/ip6tables`, `net.ipv4.ip_forward`
    - Disables swap (runtime and in `/etc/fstab`)

- Role: `containerd`
  - Installs `containerd.io` via Docker apt repo
  - Generates `/etc/containerd/config.toml`
  - Ensures `SystemdCgroup = true` and CRI plugin enabled

- Role: `kube_tools`
  - `k8s_minor_version` (default: `v1.34`)
  - Adds Kubernetes apt repo for the minor version and installs `kubelet`, `kubeadm`, `kubectl`
  - Holds package versions and enables (but does not start) `kubelet`

- Role: `kubeadm_init`
  - `k8s_cluster_name` (default: `"main-cluster"`)
  - `k8s_pod_network_cidr` (default: `10.244.0.0/16`) — matches Flannel defaults
  - `k8s_install_cni` (default: `true`)
  - `k8s_cni_manifest_url` (default: Flannel URL)
  - `k8s_save_join_command` (default: `true`)
  - `k8s_join_command_path` (default: `/etc/kubernetes/kubeadm_join.sh`)
  - After initialization, copies `/etc/kubernetes/admin.conf` to the Ansible user’s `$HOME/.kube/config`
  - Optionally installs Local Path Provisioner and sets it as the default StorageClass

## Usage

All commands are run from the project root. `ansible.cfg` already points to the inventory.

1) Initialize/prepare hosts (hostname, prerequisites, containerd, Kubernetes packages):

```
ansible-playbook playbooks/init.yml
```

2) Initialize the Kubernetes control plane:

```
ansible-playbook playbooks/kubeadm_init.yml
```

Notes:
- Ensure your inventory has at least one host in the `control_plane` group.
- The `kubeadm_init` role uses `--pod-network-cidr={{ k8s_pod_network_cidr }}`; adjust via role defaults/vars to match a different CNI if needed.
- If `k8s_save_join_command` is true, the join command is saved at `k8s_join_command_path` on the control plane node.

## Environment and Variables

This project primarily uses Ansible inventory and role variables; there is no `.env` file. Customize behavior via:

- Inventory host/group vars (e.g., SSH user, key path)
- Extra vars at runtime, for example:

```
ansible-playbook playbooks/kubeadm_init.yml \
  -e k8s_pod_network_cidr=10.244.0.0/16 \
  -e k8s_cni_manifest_url=https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

## Scripts and Entry Points

- Entry points are the playbooks:
  - `playbooks/init.yml`
  - `playbooks/kubeadm_init.yml`

No custom shell scripts are included in this repo; however, the `kubeadm_init` role can generate a join script on the target host at `k8s_join_command_path`.

## Tests

Automated tests are not present in this repository.

TODO:
- Add Molecule scenarios for roles (`containerd`, `k8s_prereqs`, `kube_tools`, `kubeadm_init`, `hostname_sync`).
- Add CI workflow to lint (`ansible-lint`, `yamllint`) and test roles.

## Troubleshooting

- If `kubectl` commands fail from your control machine, ensure the kubeconfig was copied for the Ansible user on the control plane node (`~/.kube/config`) or fetch `/etc/kubernetes/admin.conf` manually.
- Verify swap is disabled on all nodes: `swapon --show` should be empty; `/etc/fstab` should have swap entries commented.
- Confirm `containerd` is running and `SystemdCgroup = true` in `/etc/containerd/config.toml`.

## License

No license file detected.

TODO: add a `LICENSE` file and specify the project license.

## Notes

- Current date/time (for context, e.g., when defaults were last reviewed): 2025-12-07 16:25

