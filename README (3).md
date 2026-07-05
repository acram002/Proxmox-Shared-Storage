# Proxmox Cluster — Shared Storage Architecture

**Network File Share Setup & Access Control**

_Prepared: June 2026_

---

## 1. Overview

This document describes the shared network storage architecture configured across a 3-node Proxmox VE cluster. Storage is hosted on the node **iolite** (`192.168.68.91`), which has a dedicated NVMe SSD mounted at `/mnt/backups`. The storage is exposed via **NFS** to all Proxmox nodes and via **Samba (CIFS)** to Windows and Linux VMs.

---

## 2. Infrastructure

### Proxmox Cluster Nodes

| Node | IP Address | Role | Storage |
|---|---|---|---|
| iolite | 192.168.68.91 | Storage Host | NVMe SSD (nvme1n1) |
| carnelian | 192.168.68.71 | Compute Node | NFS/CIFS client |
| (3rd node) | 192.168.68.x | Compute Node | NFS/CIFS client |

### Storage Hardware

- **Device:** NVMe SSD (`nvme1n1`) on node iolite
- **Filesystem:** ext4
- **Mount point:** `/mnt/backups` (via UUID in `/etc/fstab`)
- **Network:** All nodes share the same Linux bridge (subnet `192.168.68.0/22`)

---

## 3. Directory Structure

Four subdirectories are provisioned under `/mnt/backups`, each with distinct permissions and access protocols:

| Directory | Protocol | Access | Purpose |
|---|---|---|---|
| `/mnt/backups/veeam-repo` | Samba only | veeamuser, veeam-workers | Veeam Backup Repository |
| `/mnt/backups/proxmox-storage` | NFS only | All 3 Proxmox nodes | VM disk images & containers |
| `/mnt/backups/iso-storage` | NFS only | All 3 Proxmox nodes | ISO images & CT templates |
| `/mnt/backups/shared-data` | Samba + NFS | shared-data group / subnet | General shared file access |

---

## 4. NFS Configuration

NFS is used for Proxmox node-to-node storage access. The following exports are configured on iolite in `/etc/exports`:

```
/mnt/backups/proxmox-storage  192.168.68.0/22(rw,sync,no_subtree_check,no_root_squash)
/mnt/backups/iso-storage      192.168.68.0/22(rw,sync,no_subtree_check,no_root_squash)
/mnt/backups/shared-data      192.168.68.0/22(rw,sync,no_subtree_check,no_root_squash)
```

> **Note:** `no_root_squash` is required on `proxmox-storage` and `iso-storage` as Proxmox performs storage operations as root. All nodes also mount `/mnt/shared-data` locally via NFS for direct node-level file access.

---

## 5. Proxmox Cluster Storage (GUI)

The following storage entries are configured at the Datacenter level in the Proxmox GUI and are automatically available to all nodes:

| Storage ID | NFS Export | Content Types | Nodes |
|---|---|---|---|
| `cluster-storage` | `/mnt/backups/proxmox-storage` | Disk image, Container | All |
| `cluster-isos` | `/mnt/backups/iso-storage` | ISO image, CT template | All |

---

## 6. Samba Configuration

Samba is configured on iolite for Windows VM and Linux VM access. Two shares are exposed:

| Share | Path | Valid Users | Notes |
|---|---|---|---|
| `veeam-repo` | `/mnt/backups/veeam-repo` | veeamuser, @veeam-workers | Hidden (`browseable=no`), restricted to Veeam |
| `shared-data` | `/mnt/backups/shared-data` | @shared-data group | Accessible by VMs with credentials |

---

## 7. VM Access to shared-data

### Windows VMs

- Map network drive to: `\\192.168.68.91\shared-data`
- Authenticate with a Samba user account in the `shared-data` Linux group
- Full read/write access once connected

### Linux VMs

- Install `cifs-utils` package
- Mount via:
  ```
  mount.cifs '//192.168.68.91/shared-data' /mnt/shared-data -o username=shareuser,uid=1000,gid=1000
  ```
- Use a credentials file at `/etc/samba/shared-data.creds` (`chmod 600`) to avoid hardcoding passwords
- Add entry to `/etc/fstab` with `_netdev` option for persistent mounts
- Alternatively mount via NFS if the VM IP is within `192.168.68.0/22`

---

## 8. Security Model

| Path | Protocol | Who Can Access | Root Access |
|---|---|---|---|
| `veeam-repo` | SMB only | veeamuser, @veeam-workers | No |
| `proxmox-storage` | NFS only | Node IPs only | Yes (required) |
| `iso-storage` | NFS only | Node IPs only | Yes (required) |
| `shared-data` | SMB + NFS | @shared-data group / subnet | No (squashed) |

The `veeam-repo` share is not browseable on the network and is restricted to specific Samba user accounts. Proxmox storage shares are not exposed via Samba and are inaccessible to VMs directly. Firewall rules restrict NFS (port 2049) and Samba (port 445) to the local subnet only.
