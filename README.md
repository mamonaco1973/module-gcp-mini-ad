# GCP Mini Active Directory Module

This Terraform module provisions a **Mini Active Directory (Mini-AD)** environment in **Google Cloud Platform (GCP)** using **Samba 4** on Ubuntu.  
It is designed for lab, development, and testing scenarios where a lightweight but functional Active Directory Domain Controller is required.

---

## 📖 Features

- Provisions an **Ubuntu VM** in a specified VPC/subnet  
- Bootstraps **Samba 4** as an **AD Domain Controller** and DNS server  
- Configures **Kerberos realm** and **NetBIOS** domain  
- Manages AD users via a **JSON definition file**  
- Includes a **UID/GID allocation service** (`maxUidNumber.py` + `maxids.service`)  
- Outputs key connection and identity details for downstream modules  

---

## 📂 Module Structure

- **`dc.tf`** – Core resources for VM, firewall, and AD configuration  
- **`variables.tf`** – Input variable definitions  
- **`outputs.tf`** – Exposed values such as VM IPs and admin credentials  
- **`scripts/mini-ad.sh.template`** – Bootstraps Samba AD on the VM  
- **`scripts/users.json.template`** – Defines initial user accounts  
- **`scripts/maxUidNumber.py`** – Helper to assign sequential UIDs/GIDs  
- **`scripts/maxids.service`** – Systemd service wrapper for ID allocation  

## 📥 Inputs

| Name               | Type    | Example Value                           | Description |
|--------------------|---------|-----------------------------------------|-------------|
| `netbios`          | string  | `"MCLOUD"`                              | NetBIOS short domain name (≤15 chars, uppercase). Used by legacy clients and SMB/NetBIOS workflows. |
| `realm`            | string  | `"MCLOUD.MIKECLOUD.COM"`                | Kerberos realm (uppercase DNS zone). Required by Samba/Kerberos configuration. |
| `dns_zone`         | string  | `"mcloud.mikecloud.com"`                | DNS zone and AD domain FQDN. This becomes the AD DNS namespace. |
| `user_base_dn`     | string  | `"CN=Users,DC=mcloud,DC=mikecloud,DC=com"` | Base DN in LDAP tree where user accounts are created. |
| `users_json`       | string (JSON) | [JSON blob](./scripts/users.json.template) (from `users.json.template`) | JSON definition of AD users (username, password, uid/gid). Used to pre-seed domain users. |
| `ad_admin_password`| string  | `random_password.admin_password.result` | Administrator password for the AD domain (`Administrator@REALM`). |
| `network`          | string (resource ID) | `google_compute_network.ad_vpc.id` | GCP VPC network where the AD VM is deployed. |
| `subnetwork`       | string (resource ID) | `google_compute_subnetwork.ad_subnet.id` | GCP subnet to host the AD VM. |
| `email`            | string  | Service account email                   | Service account email assigned to the VM for API/secret access. |
| `machine_type`     | string  | `"e2-medium"`                           | GCP machine type for the VM running Samba AD. Determines CPU/memory sizing. |


## 🚀 Usage Example

```hcl
# ==========================================================================================
# Mini Active Directory (mini-ad) Module Invocation
# ------------------------------------------------------------------------------------------
# Provisions an Ubuntu-based AD Domain Controller in GCP with custom realm and users
# ==========================================================================================

module "mini_ad" {
  source            = "github.com/mamonaco1973/module-gcp-mini-ad"

  # Active Directory naming
  netbios           = "MCLOUD"                                   # NetBIOS short domain
  realm             = "MCLOUD.MIKECLOUD.COM"                     # Kerberos realm (UPPERCASE)
  dns_zone          = "mcloud.mikecloud.com"                     # DNS zone / AD domain FQDN

  # Networking
  network           = google_compute_network.ad_vpc.id           # Target VPC
  subnetwork        = google_compute_subnetwork.ad_subnet.id     # Target subnet

  # Identity & authentication
  users_json        = local.users_json                           # JSON blob for users
  user_base_dn      = "CN=Users,DC=mcloud,DC=mikecloud,DC=com"   # LDAP base DN
  ad_admin_password = random_password.admin_password.result      # Randomized admin password

  # VM settings
  email             = local.service_account_email                # Service account email for VM
  machine_type      = "e2-medium"                                # Example machine type for AD VM
}
