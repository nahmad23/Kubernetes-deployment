# Kubernetes on KVM (uxus1kub1ctl01p / uxus1kub1ctl02p)

Ansible automation that creates the cluster VMs on the two KVM hosts with your existing `ulx.ops` roles, configures them, and builds an HA kubeadm cluster: 3 control planes behind your HAProxy + keepalived API VIP, 6 workers, containerd and Calico.

It is a refactor of your existing provisioning playbook. The four copy-pasted "Create ... on ctlXXp" blocks are replaced by one loop driven by `inventory/hosts.yml`, so adding or moving a VM is a one-line inventory change.

## Why Ansible rather than Terraform

- **One tool end to end.** Terraform only creates VMs. It can't install containerd, run `kubeadm` or join workers, so you'd still need Ansible.
- **You already have the KVM roles.** `ulx.ops.kvm_vm_provision`, `k8s_api_lb` and friends work today. Terraform's libvirt provider would mean rewriting VM creation for no gain.
- **No state file.** The playbooks check what exists and only change what's missing, so there's no `tfstate` to store, lock and back up.

## What runs, in order

| Playbook | Runs on | Does |
|---|---|---|
| `01-provision-vms.yml` | KVM hosts | `ulx.ops.kvm_install`, `ulx.ops.network`, then `ulx.ops.kvm_vm_provision` for every VM whose `kvm_host` is that host (two NICs, static IPs, Longhorn passthrough disks) |
| `02-base-config.yml` | all VMs (as `ubuntu`) | Your existing plays: SSH keys, `ansible` user + sudo, sshd_config, inotify/nofile limits |
| `03-prepare-nodes.yml` | all VMs | Swap off, kernel modules, sysctls, `/etc/hosts`, containerd (systemd cgroups), kubelet/kubeadm/kubectl (held), kubelet pinned to the 10.73.5.x NIC |
| `04-api-lb.yml` | control planes | Your `ulx.ops.k8s_api_lb`: VIP 10.71.251.26, HAProxy :8443 → apiservers :6443 |
| `05-control-plane.yml` | control planes, one at a time | `kubeadm init` on ctr001, `kubeadm join --control-plane` on ctr002/003, `admin.conf` copied to `artifacts/` |
| `06-cni.yml` | ctr001 | Calico via the Tigera operator, node addresses taken from 10.73.5.0/24 |
| `07-join-workers.yml` | workers | `kubeadm join`, worker label, waits for every node Ready |
| `site.yml` | | Runs 01 to 07 |
| `destroy-vms.yml` | KVM hosts | Stops and undefines the VMs (needs `-e confirm_destroy=yes`); leaves disk images and Longhorn LUNs alone |

Every step is safe to re-run.

## Networks

| | NIC | Bridge | Subnet | Used for |
|---|---|---|---|---|
| Management | `enp1s0` | br0 | 10.71.251.0/24, gw .254 | SSH/Ansible, default route, API VIP .26 |
| Kubernetes | `enp2s0` | br1 | 10.73.5.0/24 | kubelet node IP, etcd, apiserver advertise, Calico |

Because the default route is on `enp1s0`, Kubernetes would otherwise pick the management IP. The automation pins it to `enp2s0` in three places: kubelet `--node-ip`, kubeadm `advertiseAddress`, and Calico `nodeAddressAutodetectionV4`.

The API name `lb-apiserver.kubernetes.ulx.com` is added to `/etc/hosts` on every VM, both at first boot (cloud-init) and by `03-prepare-nodes.yml`, since there's no DNS for it.

## Layout

```
inventory/
  hosts.yml                    # KVM hosts + every VM: mgmt IP, k8s IP, KVM host, passthrough disks
  group_vars/all/main.yml      # networks, API VIP/name/port, k8s version, CIDRs, Calico version
  group_vars/all/vault.yml     # root password, keepalived password (see vault.yml.example)
  group_vars/kvm_hosts/        # cloud image passed to kvm_vm_provision
  group_vars/k8s_cluster/      # SSH user, default VM size (8 vCPU, 48000 MB, 200G)
  group_vars/kube_node/        # worker packages (nfs-common)
roles/
  k8s_common/  k8s_control_plane/  k8s_cni/  k8s_worker/
playbooks/
artifacts/                     # admin.conf (git-ignored)
```

Group names (`k8s_cluster`, `kube_control_plane`, `kube_node`) match your existing playbooks, so they still work against this inventory.

## Before the first run

- Make `ulx.ops` available (add its source to `requirements.yml`, or keep it on your collections path).
- Put `ulx_ssh_authorized_keys` in `group_vars/all` and `vault_kvm_vm_root_password` / `vault_k8s_api_keepalived_auth_pass` in an encrypted `vault.yml`, or point at your existing ones.
- Check that `ulx.ops.kvm_vm_provision` accepts netplan `routes:`. The draft uses `routes: [{to: default, via: ...}]` instead of `gateway4`, which netplan has deprecated.

```bash
ansible-playbook playbooks/site.yml --ask-vault-pass
export KUBECONFIG=$PWD/artifacts/admin.conf && kubectl get nodes -o wide
```

## Changes from your existing code

- **bootcmd** used to append to `/etc/hosts` on every boot, which gave duplicate lines. It also pointed the API name at ctr001 (10.71.251.13), so if ctr001 went down nobody could reach the API. It now adds the line once and points at the VIP 10.71.251.26.
- **sudoers**: the rule now goes in `/etc/sudoers.d/90-ansible` instead of editing `/etc/sudoers`.
- **sshd_config**: it is now checked with `sshd -t` before being written, and ssh only restarts when the file changes. Before, a bad config could lock you out, and ssh restarted on every run. The content is unchanged. Note that it still allows `hmac-sha1`, which is worth dropping if no old clients need it.
- **Hard-coded VM lists and IPs** have moved into the inventory.

## Limits

- **Etcd quorum.** ctl01p runs two control planes (ctr001, ctr003). If that host goes down, etcd loses quorum and the API stops, though running pods keep going. Losing ctl02p is fine. True host-level HA needs a third host.
- **Upgrades** are not automated. Bump `k8s_version_minor`, then run `kubeadm upgrade` node by node.
