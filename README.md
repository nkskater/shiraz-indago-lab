# shiraz-indago-lab

A nested-Proxmox mock of the DMW&H **indaGO Shiraz** server environment, built and operated with
Ansible. The vendor reference design is two VMware hosts with shared storage; this lab rebuilds
that shape on Proxmox VE to practise the build and to write migration and failover runbooks.

```
proxmox (physical, 192.168.1.10)
├── shiraz-pve-a  192.168.1.31  ┐ nested PVE, 2-node cluster "shiraz-lab"
├── shiraz-pve-b  192.168.1.32  ┘ ZFS + replication, HA
├── shiraz-qdevice 192.168.1.33   quorum tiebreaker (LXC)
│
└── inside the nested cluster:
    ├── shiraz-db   192.168.1.35  Ubuntu 24.04, SQL Server 2022 container
    └── shiraz-app  192.168.1.36  Ubuntu 24.04, Docker compose stack
```

Design notes, the mapping to the vendor spec, the runbook and known limitations are in
[docs/shiraz-mock.md](docs/shiraz-mock.md).

## Build status

| Phase | Playbook | State |
|---|---|---|
| 1 Nested hosts | `playbooks/shiraz/01-nested-hosts.yml` | written, not yet run |
| 2 Cluster + storage | `playbooks/shiraz/02-cluster.yml` | not written |
| 3 Mock servers | `playbooks/shiraz/03-mock-servers.yml`, `site.yml` | not written |
| 4 Runbooks | `migrate`, `drain-host`, `failover-test`, `maintenance`, `verify`, `destroy` | not written |

## Requirements on the control node

- ansible-core 2.16 or newer (built with 2.21), installed with pipx
- SSH key `~/.ssh/ansible` (its public half is put on every lab machine by cloud-init)

```bash
ansible-galaxy collection install -r collections/requirements.yml
pipx runpip ansible install -r requirements.txt
```

## Secrets

All secrets are in `inventory/group_vars/all/vault.yml`, encrypted with vault-id `shiraz`.
The vault password is read from `~/.config/ansible/shiraz_vault_pass` (mode 0600, outside the
repo, not shared with any other project). `docs/vault.example.yml` lists the keys.

```bash
ansible-vault view inventory/group_vars/all/vault.yml   # read a password
ansible-vault edit inventory/group_vars/all/vault.yml   # change one
```

## One-time setup on the physical node

Run as root on `proxmox`. This creates the lab pool and a least-privilege API token whose guest
permissions exist only on `/pool/shiraz-mock`, so Ansible cannot stop, change or delete anything
outside the lab.

```bash
pveum pool add shiraz-mock --comment "Nested PVE lab - shiraz-indago-lab"
pveum role add ShirazLabProvisioner --privs "VM.Allocate VM.Clone VM.Audit VM.PowerMgmt VM.Console VM.Config.CDROM VM.Config.CPU VM.Config.Cloudinit VM.Config.Disk VM.Config.HWType VM.Config.Memory VM.Config.Network VM.Config.Options VM.GuestAgent.Audit Pool.Audit"
pveum user add ansible-shiraz@pve --comment "shiraz-indago-lab Ansible"
pveum acl modify /pool/shiraz-mock --users ansible-shiraz@pve --roles ShirazLabProvisioner
pveum acl modify /storage/zfspool --users ansible-shiraz@pve --roles PVEDatastoreUser
pveum acl modify /storage/local --users ansible-shiraz@pve --roles PVEDatastoreUser
pveum acl modify /sdn/zones/localnetwork/vmbr0 --users ansible-shiraz@pve --roles PVESDNUser
pveum acl modify /nodes/proxmox --users ansible-shiraz@pve --roles PVEAuditor
pveum user token add ansible-shiraz@pve ansible --privsep 0
```

The last command prints the token secret once. Put it in the vault in place of `CHANGE_ME`:

```bash
ansible-vault edit inventory/group_vars/all/vault.yml   # vault_physical_api_token_secret
```

## Running

```bash
# Phase 1: Debian template, two nested VMs, Proxmox VE installed on both
ansible-playbook playbooks/shiraz/01-nested-hosts.yml

# Only one stage
ansible-playbook playbooks/shiraz/01-nested-hosts.yml --tags preflight
ansible-playbook playbooks/shiraz/01-nested-hosts.yml --tags template
ansible-playbook playbooks/shiraz/01-nested-hosts.yml --tags provision
ansible-playbook playbooks/shiraz/01-nested-hosts.yml --tags configure

# Preview changes on hosts that already exist
ansible-playbook playbooks/shiraz/01-nested-hosts.yml --check --diff
```

`--check` cannot preview the very first build: the API modules do not support check mode and the
hosts to configure do not exist yet.

## Layout

```
ansible.cfg                 inventory, roles path, vault-id, lab-only known_hosts file
inventory/hosts.yml         physical, nested_pve, qdevice, mock_servers
inventory/group_vars/       all/ (shared + vault), physical, nested_pve
inventory/host_vars/        per-host VMID
playbooks/shiraz/           numbered build playbooks, then runbook playbooks
roles/pve_cloud_template/   cloud image -> Proxmox template (qm, root on the node)
roles/pve_guest/            clone + cloud-init + disks through the API, create-if-absent
roles/pve_host/             Proxmox VE on Debian 13: repos, kernel, bridge, sshd
docs/shiraz-mock.md         architecture, spec mapping, runbook, limitations
```
