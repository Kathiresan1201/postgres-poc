# Morpheus Catalog POC: PostgreSQL on vSphere

One catalog order runs two steps:

1. **Provision VM.** Morpheus clones an Ubuntu template in vCenter, using the group, cloud, network, size and disk size picked in the order form.
2. **Install PostgreSQL.** A workflow runs [`morpheus_site.yml`](../morpheus_site.yml), which applies `roles/postgresql` to the new VM.

These are fixed for the POC: PostgreSQL 16, the role's default tuning, the `postgres` superuser reachable only locally, and no UFW changes. The app user can connect from **any** network (password required, app database only). Before any non-POC use, narrow `postgresql_allowed_networks` in `morpheus_site.yml` to the application subnet.

## Prerequisites

- A vSphere cloud in Morpheus, with an Ubuntu 24.04 template registered as a Virtual Image. The POC uses **Morpheus Ubuntu 24.04 20250218** (image 323): a clean image with a 5 GB minimum disk, cloud-init and the agent.
- VM size of at least 4 GB RAM, because the role defaults to 1 GB `shared_buffers`. The smallest plan offered is 1 vCPU / 4 GB.
- A cloud-init user for SSH. The image has no stored credentials, so Morpheus uses the Linux user from the ordering user's *User Settings*.
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
For the POC, these are **Manual** lists of known-good vCenter values, so every combination is valid. The dataset is shown for this lab; replace the IDs with your own.

| Option List | Dataset |
|---|---|
| `POC Groups` | `[{"name":"vcenter","value":"1"}]` |
| `POC Clouds` | `[{"name":"vcenter","value":"4"}]` |
| `POC Networks` | `[{"name":"VM-workload","value":"101"},{"name":"VM Network","value":"83"}]` |
| `POC Plans` | `[{"name":"1 vCPU / 4 GB","value":"220"},{"name":"2 vCPU / 8 GB","value":"222"},{"name":"2 vCPU / 16 GB","value":"224"}]` |

**5. Inputs:** *Library › Options › Inputs*. The Field Name must match exactly. Make all of them required.

| Step | Label | Field Name | Type | Option List / Default |
|---|---|---|---|---|
| 1 – VM | Group | `pocGroup` | Select List | `POC Groups`, default `1` |
| 1 – VM | Cloud | `pocCloud` | Select List | `POC Clouds`, default `4` |
| 1 – VM | Network | `pocNetwork` | Select List | `POC Networks`, default `101` |
| 1 – VM | VM Size | `pocPlan` | Select List | `POC Plans`, default `220` |
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
- **The dropdowns are fixed lists.** To offer more groups, clouds, networks or plans, add entries to the Manual option lists. To list them dynamically, switch those lists to Morpheus Api type, but those lists aren't filtered: Plans, for example, would include every AWS plan.
- **Disk size is the root disk,** and PostgreSQL data lives there (`/var/lib/postgresql`). It must be at least as large as the template's disk.

## Test

1. Order the item from *Provisioning › Catalog*.
2. Watch the instance's **History** tab. The Ansible output ends with the install summary.
3. From any host that can reach the VM, connect:
   ```bash
   psql -h <vm-ip> -U appuser -d appdb
   ```

If the playbook fails with `customOptions is undefined`, Morpheus is passing the inputs under a different name. Add a `debug: var=vars` task to see the actual structure, then adjust `morpheus_site.yml`.
