# Nexus — Ubuntu Home Infrastructure Server

**Linux administration · Self-hosting · Network services · Storage · Backup verification · Security review**

> **Project status:** Active home lab. This is a practical, guided learning project that I have configured, maintained, and troubleshot with assistance. The inventory and findings below are based on commands and logs observed on the system on **November 11, 2026**. I distinguish verified behavior from planned improvements.

## Project overview

Nexus is a repurposed PC running Ubuntu Server. I use it to practice Linux administration and host services for my home network, including containerized applications, file sharing, and network-attached backup storage.

The project lets me work with real operating-system, networking, storage, and service-management tasks rather than simulations alone. I am documenting what I can verify, what I still need to test, and what I learn as I go.

### Goals

- Maintain a working Ubuntu server and learn its day-to-day administration.
- Understand Docker applications and their network/storage dependencies.
- Support access to home services over a private network.
- Provide network storage for Proxmox backups.
- Audit access controls and address security issues without disrupting services.
- Practice explaining and troubleshooting the environment independently.

## Architecture at a glance

```text
                 Home LAN (private IPv4 /24)
                            |
                 Router / managed switch
                            |
          +-----------------+------------------+
          |                                    |
  Nexus — Ubuntu Server                pve1 — Proxmox host
  - Docker containers                  - LXC container 100 (Pi-hole)
  - Nextcloud stack                    - LXC container 101
  - Samba file share                   - VM 102 (older backup found)
  - NFS backup export     <--- NFS ---  - Scheduled vzdump backups
  - Tailscale access
```

*Simplified logical view. This is **not** a verified physical cabling diagram or proof of VLAN isolation. Identifiers and sensitive endpoints have been omitted.*

## Verified system inventory

| Category | Observed configuration |
| --- | --- |
| Hostname | `nexus` |
| Operating system | Ubuntu Server 26.04.1 LTS (reported from `hostnamectl`) |
| Kernel | `7.0.0-43-generic` (reported from `hostnamectl`) |
| LAN addressing | Private IPv4 `/24` network |
| Remote connectivity | Tailscale interface observed |
| Containers | Docker with bridge networks observed |
| Main OS storage | 100 GB ext4 root logical volume (LVM) |
| Additional LVM storage | 363.8 GB ext4 mounted at `/srv/storage` |
| Additional disk | ~3.6 TB ext4 mounted at `/srv/storage4tb` |
| Separate boot filesystem | 2 GB ext4 mounted at `/boot` |

The `lsblk` output also showed a ~116 GB VFAT volume; its purpose and mount status have **not** been established. Hardware model, filesystem redundancy, and disk-health status remain to be documented.

### Observed container images

The following images were reported as running during the review:

- `nextcloud:34.0-apache`
- `nextcloud:apache`
- `redis:alpine`
- `mariadb:lts`
- `portainer/portainer-ce:lts`

**Open question:** Two Nextcloud images appeared in the container list. Their exact container names, roles, and whether both are intended to be running still need verification. Redis and MariaDB were observed but their individual application connections were not traced during this review.

## Backup workflow and validation

Nexus exports a directory over NFS for Proxmox backup storage:

```text
Nexus: /srv/storage/backups/proxmox
                      └── dump/
```

The export is restricted to one designated local Proxmox client at the UFW layer. Proxmox writes `vzdump` archives and related log/notes files into the `dump` directory.

### Backup evidence — November 11, 2026

| Workload | Method | Data processed | Archive size | Elapsed time | Log result |
| --- | --- | ---: | ---: | --- | --- |
| LXC 100 — Pi-hole | Snapshot | ~1.1 GiB | 364 MB | 1m 03s | Finished backup |
| LXC 101 — name not yet confirmed | Snapshot | ~1.1 GiB | 331 MB | 1m 05s | Finished backup |

Both logs reported **`keep-last=7`** retention and the pruning of one older backup. A previous backup archive for VM 102 was found dated October 9, 2026; whether VM 102 still requires routine backups has not been confirmed.

**Important limitation:** The logs verify successful backup job completion and archive creation, **not** successful restoration. I have not yet documented a restore test, independent integrity verification, or an off-site backup copy.

### Read-only commands used to verify backups

```bash
sudo ls -lah /srv/storage/backups/proxmox/dump
sudo tail -n 20 /srv/storage/backups/proxmox/dump/<backup-log>.log
```

## Security hardening and audit

This is an **audit of existing configuration**, not a claim that the server is fully hardened. The items below were inspected with read-only commands during a guided troubleshooting session.

| Area | Verified finding | Assessment / follow-up |
| --- | --- | --- |
| UFW | Firewall reported active | Default incoming policy not yet documented |
| NFS RPCbind (`111/tcp`, `111/udp`) | Allow from one designated LAN client, then deny other sources | Source-scoped rules present; effective reachability still needs testing |
| NFS (`2049/tcp`, `2049/udp`) | Allow from the same designated LAN client, then deny other sources | Verify from both authorized and unauthorized clients |
| SMB (`445/tcp`) | Allow from the local `/24` subnet | Validate against intended access and interface exposure |
| IPv6 NFS ports | Deny rules for ports 111 and 2049 appeared | Further inspection recommended for all relevant services |
| NFS export | `rw,sync,sec=sys,no_root_squash` for one client | **Risk identified:** review `no_root_squash` before changing |
| Export filesystem permissions | `root:root`, mode `755` on export and `dump/` | Explains possible dependency on unsquashed remote root for writes |
| Remote access | Tailscale address/interface observed | ACLs, SSH settings, and reachability not fully reviewed |

### Security finding: `no_root_squash`

The NFS export presently permits root on its authorized client to act as root against the exported directory. This can increase the impact of a compromised or misconfigured client. The export also uses `sec=sys`, which relies on traditional client-provided UNIX identity information rather than strong per-user cryptographic authentication.

I have **not** changed this setting. Both the export directory and `dump/` were owned by `root:root` with permissions `755`, so switching directly to `root_squash` may prevent Proxmox from writing backups. A safe change requires checking the Proxmox storage configuration, testing ownership/mapping and write access, planning rollback, and re-verifying backup **and restore** behavior.

### Read-only audit commands

```bash
hostnamectl
ip -br addr
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS
sudo ufw status numbered
sudo ufw show added
sudo exportfs -v
sudo ls -ld /srv/storage/backups/proxmox
sudo ls -lah /srv/storage/backups/proxmox/dump
```

*Actual IP addresses, private endpoints, and raw logs are intentionally omitted from this public document. Commands are included for learning and repeatable verification; they are not instructions to apply firewall changes to another system.*

## Lessons and skills being developed

- Reading Linux host, interface, storage, and service inventory commands.
- Recognizing local versus container and private overlay-network addressing.
- Understanding the relationship between host file permissions, NFS exports, and remote client identities.
- Using UFW rule order and source restrictions to reason about network access.
- Interpreting backup logs, snapshot mode, archive sizes, and retention behavior.
- Separating successful job completion from validated disaster recovery.
- Making cautious changes to a working environment with a test and rollback plan.

## Next steps

- [ ] Identify the exact roles of both Nextcloud containers and document the application/data layout.
- [ ] Confirm UFW default policies and test NFS/SMB exposure from intended and unintended clients.
- [ ] Verify NFS export configuration and evaluate a safer root-mapping approach with a tested rollback.
- [ ] Check Proxmox storage ownership and test a **non-production** backup and restore workflow.
- [ ] Determine whether VM 102 is still active and should be covered by the backup schedule.
- [ ] Create a sanitized physical network diagram and explain which segments are truly isolated.
- [ ] Add evidence of a troubleshooting task I can reproduce and explain end-to-end.
- [ ] Add annotated screenshots without publishing secrets, live tokens, or customer information.

## Approach and attribution

This portfolio is a record of **hands-on, guided learning**. I have used step-by-step assistance to configure and inspect parts of the environment, and I am intentionally documenting what I can demonstrate rather than claiming independent design or unverified production-level security. As I repeat tasks independently, I will update the repository with the commands, rationale, test results, and lessons learned.

---

*Personal lab environment. No customer systems, credentials, API keys, Tailscale authentication keys, or sensitive configuration files are included.*
