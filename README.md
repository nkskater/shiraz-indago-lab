# shiraz-indago-lab

A nested-Proxmox mock of the DMW&H **indaGO Shiraz** server environment, built and operated with
Ansible. The vendor reference design is two VMware hosts with shared storage; this lab rebuilds
that shape on Proxmox VE to practise the build and to write migration and failover runbooks.

```
proxmox (physical, 192.168.1.10)
├── 9101 shiraz-pve-a   192.168.1.31  ┐ nested PVE 9.2, cluster "shiraz-lab"
├── 9102 shiraz-pve-b   192.168.1.32  ┘ ZFS + replication every 5 min, HA
├── 9103 shiraz-qdevice 192.168.1.33    quorum tiebreaker (LXC)
│
└── inside the nested cluster (pool shiraz-mock):
    ├── 9201 shiraz-db   192.168.1.35  Ubuntu 24.04, SQL Server 2022 container
    └── 9202 shiraz-app  192.168.1.36  Ubuntu 24.04, Docker compose stack + SQL probe
```

Design notes, the mapping to the vendor spec, the runbook, measured results and known limitations
are in [docs/shiraz-mock.md](docs/shiraz-mock.md).

## Requirements on the control node

- ansible-core 2.16 or newer (built and tested with 2.21), installed with pipx
- SSH key `~/.ssh/ansible` (cloud-init puts its public half on every lab machine)

```bash
ansible-galaxy collection install -r collections/requirements.yml
pipx runpip ansible install -r requirements.txt
```

## Secrets

All secrets are in `inventory/group_vars/all/vault.yml`, encrypted with vault-id `shiraz`.
`inventory/group_vars/all/nested_api_token.vault.yml` is written by `02-cluster.yml` and holds
the nested cluster's API token. The vault password is read from
`~/.config/ansible/shiraz_vault_pass` (mode 0600, outside the repo, not shared with any other
project). `docs/vault.example.yml` lists the keys.

```bash
ansible-vault view inventory/group_vars/all/vault.yml   # read a password
ansible-vault edit inventory/group_vars/all/vault.yml   # change one
```

Logins: `root` on the nested hosts (web UI on `https://192.168.1.31:8006`) uses
`vault_nested_root_password`; `dmwh-admin` on the mock servers uses `vault_dmwh_admin_password`.

## One-time setup on the physical node

Run as root on `proxmox`. This creates the lab pool and a least-privilege API token whose guest
permissions exist only on `/pool/shiraz-mock`, so Ansible cannot see, stop, change or delete
anything outside the lab.

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

Put the secret that the last command prints into the vault as `vault_physical_api_token_secret`.

## Playbooks

Run everything from the repo root.

### Build

```bash
ansible-playbook playbooks/shiraz/site.yml                 # everything, in order, then verify

ansible-playbook playbooks/shiraz/01-nested-hosts.yml      # Debian template, 2 nested VMs, PVE installed
ansible-playbook playbooks/shiraz/02-cluster.yml           # cluster, QDevice, ZFS storage, token, Ubuntu template
ansible-playbook playbooks/shiraz/03-mock-servers.yml      # shiraz-db and shiraz-app, replication, HA, software
```

All three are safe to re-run and report `changed=0` on a second run. Useful tags:
`01`: `preflight`, `template`, `provision`, `configure`. `02`: `cluster`, `qdevice`, `storage`,
`access`, `template`. `03`: `preflight`, `provision`, `protect`, `configure`, `db`, `app`.

`--check --diff` works on hosts that already exist. It cannot preview a first build: the API
modules do not support check mode and the hosts to configure do not exist yet.

### Operate

```bash
# Health check: PASS/FAIL table, non-zero exit on any failure
ansible-playbook playbooks/shiraz/verify.yml

# Move one server (live is the default)
ansible-playbook playbooks/shiraz/migrate.yml -e vm=shiraz-db -e target=shiraz-pve-b
ansible-playbook playbooks/shiraz/migrate.yml -e vm=shiraz-db -e target=shiraz-pve-a -e mode=offline

# Empty a host for maintenance; optionally mark it in the HA manager
ansible-playbook playbooks/shiraz/drain-host.yml -e host=shiraz-pve-a
ansible-playbook playbooks/shiraz/drain-host.yml -e host=shiraz-pve-a -e maintenance=true
ansible-playbook playbooks/shiraz/drain-host.yml -e host=shiraz-pve-a -e maintenance=false

# Simulate a host failure (hard-stops the nested host VM)
ansible-playbook playbooks/shiraz/failover-test.yml -e confirm_failover=yes
ansible-playbook playbooks/shiraz/failover-test.yml -e confirm_failover=yes --tags recover
ansible-playbook playbooks/shiraz/failover-test.yml -e confirm_failover=yes -e failover_recover=yes -e failback=yes
ansible-playbook playbooks/shiraz/failover-test.yml -e confirm_failover=yes -e failover_host=shiraz-pve-b

# Daily maintenance window: updates, image refresh, timed against 30 minutes
ansible-playbook playbooks/shiraz/maintenance.yml
ansible-playbook playbooks/shiraz/maintenance.yml -e maintenance_reboot=never
```

### Tear down

Only guests named in the inventory are considered, and each must carry the lab's name, tag and
pool in the live Proxmox data or the run aborts.

```bash
ansible-playbook playbooks/shiraz/destroy.yml -e confirm_destroy=yes -e layer=mock      # mock servers only
ansible-playbook playbooks/shiraz/destroy.yml -e confirm_destroy=yes -e layer=nested    # nested hosts + QDevice
ansible-playbook playbooks/shiraz/destroy.yml -e confirm_destroy=yes -e layer=nested -e destroy_templates=yes
```

## Layout

```
ansible.cfg                  inventory, roles path, vault-id, lab-only known_hosts file
inventory/hosts.yml          physical, nested_pve, qdevice, shiraz_db, shiraz_app, mock_servers
inventory/group_vars/        all/ (shared + vaults) and one file per group
inventory/host_vars/         per-host VMID and home node
playbooks/shiraz/            numbered build playbooks, runbook playbooks, shared tasks/
roles/
  pve_cloud_template/        cloud image -> Proxmox template (qm, root on the node)
  pve_guest/                 clone + cloud-init + disks through the API, create-if-absent
  pve_container/             LXC through the API, create-if-absent
  pve_host/                  Proxmox VE on Debian 13: repos, kernel, bridge, sshd
  pve_cluster/               create the cluster, join nodes
  pve_qdevice/               corosync-qnetd witness + node side
  pve_zfs_storage/           ZFS pool on the data disk, ARC limit, storage entry
  pve_api_access/            pool, pool-scoped API user and token
  pve_replication/           storage replication jobs and HA resources
  shiraz_common/             spec baseline: timezone, power management, dmwh-admin, sshd, UFW
  docker/                    Docker Engine + compose plugin from Docker's apt repository
  shiraz_db/                 SQL Server 2022 container, indago_mock, shiraz_svc
  shiraz_app/                compose stack: nginx, Postgres, Adminer, SQL probe
docs/shiraz-mock.md          architecture, spec mapping, runbook, results, limitations
```
