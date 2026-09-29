# Morpheus Catalog POC: PostgreSQL on vSphere

One catalog order runs two steps:

1. **Provision VM.** Morpheus clones an Ubuntu template in vCenter, using the group, cloud, network, size and disk size picked in the order form.
2. **Install PostgreSQL.** A workflow runs [`morpheus_site.yml`](../morpheus_site.yml), which applies `roles/postgresql` to the new VM.

These are fixed for the POC: PostgreSQL 16, the role's default tuning, the `postgres` superuser reachable only locally, and no UFW changes. The app user can connect from **any** network (password required, app database only). Before any non-POC use, narrow `postgresql_allowed_networks` in `morpheus_site.yml` to the application subnet.

## Prerequisites

- A vSphere cloud in Morpheus, with an Ubuntu 22.04/24.04 template registered as a Virtual Image. It needs at least 4 GB RAM, because the role defaults to 1 GB `shared_buffers`.
- Internet access from the VM to `apt.postgresql.org`.
- This repo pushed to Git.
- The collections installed on the Morpheus appliance: `ansible-galaxy collection install -r requirements.yml`

## Setup

**1. Ansible integration:** *Administration › Integrations › + New › Ansible*
Set the Git URL and branch `main`. Playbooks Path `/`, Roles Path `roles`. Leave the group/host vars paths empty. Leave Command Bus off.

**2. Task:** *Library › Automation › Tasks › + Add*
Type **Ansible**, Repo = the integration above, Playbook `morpheus_site.yml`, Execute Target **Resource**.

**3. Workflow:** *Library › Automation › Workflows › + Add › Provisioning Workflow*
Platform Linux. Add the task to the **Provision** phase.

**4. Option lists:** *Library › Options › Option Lists › + Add*
These fill the VM dropdowns from what already exists in Morpheus. For each one, set Type **Morpheus Api** and pick the matching source:

| Option List | Source |
|---|---|
| `POC Groups` | Groups |
| `POC Clouds` | Clouds |
| `POC Networks` | Networks |
| `POC Plans` | Plans |

**5. Inputs:** *Library › Options › Inputs*. The Field Name must match exactly. Make all of them required.

| Step | Label | Field Name | Type | Option List / Default |
|---|---|---|---|---|
| 1 – VM | Group | `pocGroup` | Select List | `POC Groups` |
| 1 – VM | Cloud | `pocCloud` | Select List | `POC Clouds` (Dependent Field: `pocGroup`) |
| 1 – VM | Network | `pocNetwork` | Select List | `POC Networks` (Dependent Field: `pocCloud`) |
| 1 – VM | VM Size | `pocPlan` | Select List | `POC Plans` |
| 1 – VM | Disk Size (GB) | `pocDiskSize` | Number | default `50` |
| 1 – VM | VM Name | `instanceName` | Text | e.g. `pgpoc01` |
| 2 – DB | Database Name | `pgAppDatabase` | Text | e.g. `appdb` |
| 2 – DB | Database User | `pgAppUser` | Text | e.g. `appuser` |
| 2 – DB | Database Password | `pgAppPassword` | Password | at least 12 characters |

The Step 1 inputs are used only by Morpheus to build the VM. The Step 2 inputs are the only ones the playbook reads.

**6. Catalog item:** *Library › Blueprints › Catalog Items › + Add › Instance*
1. Attach the 9 inputs, in the order above.
2. Run the **Configuration Wizard** once with real selections: group, vSphere cloud, Ubuntu layout, plan, network, datastore and resource pool. Under **Automation**, pick the workflow from step 3.
3. In the generated Config JSON, replace the values the wizard wrote for these keys, keeping the key names and nesting it produced:

   | Key in Config JSON | Set to |
   |---|---|
   | instance name (`name` and/or `instance.name`) | `"<%=customOptions.instanceName%>"` |
   | group id (`group.id` / `instance.site.id`) | `<%=customOptions.pocGroup%>` |
   | cloud id (`cloud.id` / `zoneId`) | `<%=customOptions.pocCloud%>` |
   | plan id (`plan.id` / `instance.plan.id`) | `<%=customOptions.pocPlan%>` |
   | root volume size (`volumes[0].size`) | `<%=customOptions.pocDiskSize%>` |
   | network (`networkInterfaces[0].network.id`) | `"network-<%=customOptions.pocNetwork%>"` |

   Example of the edited parts:
   ```jsonc
   "zoneId": <%=customOptions.pocCloud%>,
   "instance": {
     "name": "<%=customOptions.instanceName%>",
     "site": { "id": <%=customOptions.pocGroup%> },
     "plan": { "id": <%=customOptions.pocPlan%> }
   },
   "volumes": [
     { "rootVolume": true, "name": "root", "size": <%=customOptions.pocDiskSize%>, "datastoreId": "auto" }
   ],
   "networkInterfaces": [
     { "network": { "id": "network-<%=customOptions.pocNetwork%>" } }
   ]
   ```
4. Save.

**Notes**
- **Resource pool, folder and datastore stay as the wizard set them**, and they belong to one specific vCenter. If users can pick a different vSphere cloud, remove those keys, or set the datastore to `"auto"`, so Morpheus uses that cloud's defaults.
- **The dropdowns list everything.** For example, Plans includes non-VMware plans, and Clouds includes clouds outside the chosen group. For the POC, just pick valid combinations. To narrow the lists later, add a Translation Script to each option list.
- **Disk size is the root disk,** and PostgreSQL data lives there (`/var/lib/postgresql`). It must be at least as large as the template's disk.

## Test

1. Order the item from *Provisioning › Catalog*.
2. Watch the instance's **History** tab. The Ansible output ends with the install summary.
3. From any host that can reach the VM, connect:
   ```bash
   psql -h <vm-ip> -U appuser -d appdb
   ```

If the playbook fails with `customOptions is undefined`, Morpheus is passing the inputs under a different name. Add a `debug: var=vars` task to see the actual structure, then adjust `morpheus_site.yml`.
