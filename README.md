---

## Prerequisites

- Red Hat Ansible Automation Platform 2.5+
- VMware vCenter access
- `community.vmware` collection available in your AAP execution environment
- Nginx web server hosting OVA files with `autoindex_format json` enabled
- Git repository connected to AAP as a project

---

## First Time Setup

### 1. Clone the repository

```bash
git clone <your-repo-url>
cd deploy_vsp360
```

### 2. Create site configuration

```bash
cp vars/site_config.yml.example vars/site_config.yml
```

Edit `vars/site_config.yml` with your environment values. This file is
excluded from Git via `.gitignore` and must be created on every new clone.

### 3. Create the vault file

```bash
cp ansible_vault_vars/ansible_vault.yml.example ansible_vault_vars/ansible_vault.yml
```

Edit `ansible_vault_vars/ansible_vault.yml` with your credentials, then encrypt it:

```bash
ansible-vault encrypt ansible_vault_vars/ansible_vault.yml
```

Store the vault password in AAP as a **Vault Credential** — it will be
used automatically when jobs run.

### 4. Configure Nginx autoindex

On your OVA file server, ensure Nginx is configured with JSON autoindex:

```nginx
location / {
    root /var/www/html;
    autoindex on;
    autoindex_format json;
}
```

Reload Nginx after changes:

```bash
nginx -t && systemctl reload nginx
```

Verify it is working:

```bash
curl http://<your-ova-server>/
```

Expected output:

```json
[
  { "name": "VSP360-x.x.x.ova", "type": "file", "mtime": "...", "size": 0 }
]
```

---

## AAP Setup

### Step 1 — Create the Deploy Job Template

Create a job template in AAP with the following settings:

| Setting | Value |
|---|---|
| Name | `Deploy VSP360` |
| Playbook | `deploy_vsp360.yml` |
| Credentials | Your Vault Credential |
| Inventory | `localhost` |
| Survey | Enabled (populated by the refresh job) |

> **Note the Job Template ID from the URL once created — you will need it
> in Step 2.**
>
> Navigate to the template and check the URL:
> `https://<aap-host>/#/templates/job_template/`**`58`**`/details`

---

### Step 2 — Create the Survey Refresh Job Template

Create a second job template with the following settings:

| Setting | Value |
|---|---|
| Name | `Refresh VSP360 Survey` |
| Playbook | `update_aap_survey.yml` |
| Credentials | Your Vault Credential |
| Inventory | `localhost` |

Add the following **Extra Variable**, replacing `<template_id>` with the
ID noted from Step 1:

```yaml
aap_job_template_id: <template_id>
```

> This tells the refresh job which template's survey to update. It must
> be set before the job will run successfully.

---

### Step 3 — Run the Survey Refresh Job

Run the `Refresh VSP360 Survey` job template manually for the first time.
This will:

- Connect to Nginx and discover available OVA files
- Connect to vCenter and discover datastores, clusters, and resource pools
- Create the survey on the `Deploy VSP360` template if it does not exist
- Populate all dropdown choices with live data
- Preserve any existing defaults

---

### Step 4 — Schedule the Survey Refresh Job

Add a schedule to the `Refresh VSP360 Survey` job template to keep the
survey choices up to date automatically:

| Setting | Value |
|---|---|
| Name | `Nightly Survey Refresh` |
| Start date/time | Today at 18:00 |
| Repeat frequency | Every day |
| RRULE | `RRULE:FREQ=DAILY;BYHOUR=18;BYMINUTE=0;BYSECOND=0` |

---

### Step 5 — Deploy VSP360

Launch the `Deploy VSP360` job template. The survey will prompt for:

| Question | Description |
|---|---|
| VSP360 Hostname | Hostname for the new VM |
| VSP360 IP Address | IP address for the new VM |
| OVA Base URL | Base URL of the OVA file server |
| VSP360 .ova | Select from available OVA files |
| VSP360 Profile | Deployment size: small, medium, or large |
| Datastore | Select from available vCenter datastores |
| VMware Cluster | Select from available vCenter clusters |
| Resource Pool | Select from available vCenter resource pools |
| Timezone | Select the VM timezone |
| Disk Provisioning | Select the disk provisioning type |

---

## Profile Sizing

| Profile | vCPUs | Memory | Data Disk |
|---|---|---|---|
| small | OVA default | OVA default | OVA default |
| medium | 15 | 46 GB | 535 GB |
| large | 18 | 57 GB | 735 GB |

---

## Variables Reference

### vars/site_config.yml

| Variable | Used By | Description |
|---|---|---|
| `aap_host` | update_aap_survey.yml | AAP server URL |
| `vcenter_hostname` | both | vCenter server IP or hostname |
| `vcenter_datacenter` | both | vCenter datacenter name |
| `ova_server_url` | both | Nginx OVA file server base URL |
| `vsp360_default_ip` | update_aap_survey.yml | Default IP shown in survey |
| `vsp360_netmask` | deploy_vsp360.yml | VM network mask |
| `vsp360_gateway` | deploy_vsp360.yml | VM default gateway |
| `vsp360_dns` | deploy_vsp360.yml | VM DNS server |
| `vsp360_domain` | deploy_vsp360.yml | VM domain name |
| `vsp360_ntp` | deploy_vsp360.yml | VM NTP server |

### ansible_vault_vars/ansible_vault.yml

| Variable | Description |
|---|---|
| `aap_user` | AAP admin username |
| `aap_pass` | AAP admin password |
| `vcenter_username` | vCenter username |
| `vcenter_password` | vCenter password |
| `vsp360_root_password` | VSP360 appliance root password |

---

## Notes

- `vars/site_config.yml` and `ansible_vault_vars/ansible_vault.yml` are
  excluded from Git via `.gitignore`. They must be created manually on
  every new clone or managed via AAP Extra Variables.
- The survey refresh job safely preserves existing survey defaults and
  all non-dynamic questions on every run.
- If no survey exists on the deploy template, the refresh job will create
  one automatically with sensible defaults.
- The refresh job will abort safely if no OVA files or vCenter objects
  are found, preventing the survey from being wiped.
