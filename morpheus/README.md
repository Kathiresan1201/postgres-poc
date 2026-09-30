# Morpheus Catalog POC: PostgreSQL on vSphere

One catalog order, **PostgreSQL on vSphere (POC)**, runs two steps:

1. **Provision VM.** Morpheus clones the Ubuntu 22.04 vCenter template `<UBUNTU_22_04_TEMPLATE>` (4 vCPU / 16 GB, disk size from the order form).
2. **Install PostgreSQL.** The provisioning workflow runs [`morpheus_site.yml`](../morpheus_site.yml), which grows the root filesystem to the ordered disk size and then applies `roles/postgresql`.

An order reaches **Running in about 5–6 minutes** with no manual steps. It was validated end to end on Morpheus 9.0.1: 109 of 109 QA checks passed (see [Validation](#validation)).

Fixed for the POC:
- **PostgreSQL:** version 16, the role's default tuning, `max_connections` 100.
- **Admin access:** the `postgres` superuser can only log in locally on the VM (peer auth). No UFW changes.
- **App access:** the app user can connect from **any** network, with a password, to the app database only. Before any non-POC use, narrow `postgresql_allowed_networks` in `morpheus_site.yml`.
- **IP address:** every VM gets **<VM_IP>**, because the template hard-codes it (see [Known limitations](#known-limitations)). **Only one VM can run at a time.**

## Order form

| # | Field | Field name | Default / rule |
|---|---|---|---|
| 1 | Group | `pocGroup` | vcenter |
| 2 | Cloud | `pocCloud` | vcenter |
| 3 | VM Name | `instanceName` | lowercase, `^[a-z][a-z0-9-]{1,62}$` |
| 4 | Network | `pocNetwork` | VM-workload |
| 5 | Disk Size (GB) | `pocDiskSize` | 100. Sets the VM disk, and the root filesystem is grown to match |
| 6 | Database Name | `pgAppDatabase` | appdb, `^[a-z_][a-z0-9_]{0,62}$` |
| 7 | Database User | `pgAppUser` | appuser, same pattern |
| 8 | Database Password | `pgAppPassword` | **at least 12 characters** (checked by the role, so a shorter one fails the install step) |

Group, Cloud, Network and Disk Size drive Step 1 (Morpheus). The Database fields are what the playbook reads in Step 2.

## Prerequisites

- **Morpheus nodes:** every node has **Ansible** (`apt install ansible`, which includes `community.postgresql` and `community.general`) and **`sshpass`**. Morpheus runs Ansible with password SSH.
- **vSphere:** a cloud in Morpheus with the template `<UBUNTU_22_04_TEMPLATE>` synced as a Virtual Image.
- **Image login:** on the Virtual Image, set the SSH user and password to the template's own login, and set **Install Agent** off. The template ignores Morpheus's cloud-init user.
- **Internet access:** the VM must reach the Ubuntu mirrors, `www.postgresql.org` and `apt.postgresql.org`.
- **Repo:** this repo on Git (`https://github.com/Kathiresan1201/postgres-poc`, branch `main`).

## Morpheus objects (as configured in the lab)

| Object | Name / ID | Key settings |
|---|---|---|
| Ansible integration | `postgres-poc-ansible` (2) | Git URL above, branch `main`, Playbooks Path `/`, Roles Path `roles`, no group/host vars, Command Bus **off** |
| Task | `PostgreSQL POC - Install` (15) | Type Ansible, playbook `morpheus_site.yml`, Execute Target **Resource** |
| Workflow | `PostgreSQL POC` (2) | Provisioning, Linux, task 15 in the **Provision** phase |
| Option lists | `POC Groups` / `POC Clouds` / `POC Networks` (Manual) | `vcenter=1` / `vcenter=4` / `VM-workload=101`, `VM Network=83` |
| Inputs | the 8 fields above | all required, display order as listed |
| Catalog item | `PostgreSQL on vSphere (POC)` (1) | Type Instance, config below |

Catalog item config: only the values that matter are shown. Everything else is as the Configuration Wizard generated it.

```jsonc
{
  "type": "vmware",
  "instance": { "name": "<%=customOptions.instanceName%>", "instanceType": { "code": "vmware" },
                "layout": { "id": 34 }, "plan": { "id": 227 }, "site": { "id": "<%=customOptions.pocGroup%>" } },
  "zoneId": "<%=customOptions.pocCloud%>",
  "plan": { "id": 227 },                          // "Custom VMWare"
  "servicePlanOptions": { "maxCores": 4, "coresPerSocket": 1, "maxMemory": 17179869184 },
  "config": {
    "template": 315,                              // <UBUNTU_22_04_TEMPLATE>
    "resourcePoolId": "<RESOURCE_POOL>",
    "createUser": false,                          // use the image's login, not the orderer's
    "noAgent": true                               // don't wait for the Morpheus agent
  },
  "volumes": [ { "rootVolume": true, "name": "root", "size": "<%=customOptions.pocDiskSize%>", "datastoreId": 23 } ],   // <LOCAL_DATASTORE>
  "networkInterfaces": [ { "network": { "id": "network-<%=customOptions.pocNetwork%>" },
                           "ipMode": "static", "ipAddress": "<VM_IP>" } ],
  "taskSetId": 2, "taskSetName": "PostgreSQL POC"
}
```

## Running the demo

1. **Delete any existing POC VM first,** because of the one-VM limit on <VM_IP>. Wait until it's gone from *Provisioning › Instances*.
2. **Order:** *Provisioning › Catalog › PostgreSQL on vSphere (POC)*. Fill in the VM name, DB name, DB user and a **password of 12 or more characters**.
3. **Watch the steps:** open the instance's **History** tab to see clone, network, SSH, then **execute task**. That task is the Ansible run, and its output ends with *PostgreSQL role completed successfully* and `failed=0`.
4. **Connect:**
   ```bash
   ssh ubuntu@<VM_IP>
   sudo -u postgres psql                                     # superuser, local only
   psql "host=<VM_IP> dbname=appdb user=appuser"         # app user, password prompt
   ```

## Validation

End-to-end QA of a fresh order (2026-09-30):

| Layer | Checks | Result |
|---|---|---|
| Morpheus: catalog item, workflow, task, instance, form rejecting bad names | 19 | pass |
| Role end state: every task's result on the VM, plus root ≈ 97 GB on a 100 GB disk | 66 | pass |
| Database: TCP login, SSL/scram, CRUD, wrong password / other DB / remote superuser / CREATE DATABASE / CREATE ROLE refused, logging, port reachable | 14 | pass |
| Idempotency: re-run the task from Morpheus | 2 | 0 changed, 0 failed |
| Reboot: IP, disk, service, login and settings survive | 8 | pass |

## Known limitations

- **Fixed IP.** The template hard-codes **<VM_IP>** in `/etc/netplan/00-installer-config.yaml` and ignores Morpheus's cloud-init. So every VM gets .106, and Morpheus's hostname and user settings aren't applied. Removing the netplan file and re-enabling cloud-init in the template (in vCenter) would allow parallel orders from the pool.
- **Host keys aren't pinned.** Because VMs are rebuilt on the same IP, `morpheus_site.yml` turns off SSH host-key checking.
- **Shared password.** The template password is stored on the Morpheus Virtual Image. Change it after the POC.

## Lessons from the lab (Morpheus 9.0.1)

- **Inputs arrive as `morpheus.customOptions`.** In native Morpheus Ansible, top-level `customOptions` is empty. `morpheus_site.yml` reads both.
- **Every Morpheus node needs `sshpass`** as well as Ansible.
- **`createUser: true` makes Morpheus log in as the orderer's personal Linux user,** not the image's. Keep it `false` for templates like this one.
- **The disk-size input only resizes the VM disk.** On a template that ignores cloud-init, the root filesystem stays at the template size. That's why `morpheus_site.yml` grows the partition, LVM and filesystem itself.
- **The Morpheus Ubuntu images (24.04 20250218 / 20260115) never brought up their network** on this vCenter, with DHCP, static or pool addressing. The cause is unknown and needs a vCenter console check.
- **Don't put the disk on <VSAN_DATASTORE>.** Morpheus's cloud-init ISO upload to vSAN hangs; use <LOCAL_DATASTORE>.
