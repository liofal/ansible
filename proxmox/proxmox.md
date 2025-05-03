# Proxmox VE Host Management Documentation

## Overview

This document outlines the management practices for the Proxmox Virtual Environment (PVE) host (`proxmox.liofal.net`) that serves as the foundation for our K3s cluster running in LXC containers. Management is primarily performed using Ansible.

## Architecture

- **Host:** A single Proxmox VE server (`proxmox.liofal.net`).
- **Role:** Hosts the LXC containers for the K3s cluster (controller and workers).
- **Networking:** [Details on relevant network configuration, if specific to Ansible management]
- **Storage:** [Details on relevant storage configuration, if specific to Ansible management]

## Management via Ansible

Ansible playbooks are used to automate key management tasks on the Proxmox host:

- **Package Upgrades:** `proxmox/playbook-upgrade-proxmox.yaml` handles system package updates.
- **Certificate Updates:** `proxmox/playbook-update-proxmox-certificate.yaml` manages the custom TLS certificate used by the Proxmox web UI/API.
- **Status Checks:**
    - `proxmox/playbook-check-unattended-upgrades-status.yaml` checks the status of the `unattended-upgrades` package.
- **K3s Cluster Management:** The Proxmox host is the target for Ansible playbooks managing the K3s cluster (e.g., `k3s/playbook-upgrade-k3s.yaml`). These playbooks use `pct exec` on this host to run commands inside the K3s LXC containers.

## Key Configurations

- **Custom TLS Certificate:** The host is configured to use a custom TLS certificate located in `/etc/pve/local/`. Updates require restarting the `nginx.service`.
- **Unattended Upgrades:** The status of automatic package upgrades is managed/checked via Ansible. [Current status: Not installed, based on previous checks].

## pmxcfs Considerations

- The Proxmox Cluster File System (`pmxcfs`) restricts direct file manipulation (permissions, ownership) within `/etc/pve/`.
- A specific workaround pattern is required when managing files in this directory using Ansible (copy to `/tmp`, then `shell: cp` without `-p`). Refer to `memory-bank/systemPatterns.md` for details.

## Maintenance Procedures

- **Host Upgrades:** Performed using `proxmox/playbook-upgrade-proxmox.yaml`.
- **Backups:** [Details on Proxmox host backup strategy, if any]
- **Monitoring:** [Details on Proxmox host monitoring, if any]

## Troubleshooting

Common issues related to managing the Proxmox host via Ansible and their resolutions.

- **Certificate Update Failures:** Often related to `pmxcfs` restrictions. Ensure the correct workaround pattern is used and `nginx.service` is restarted.
- **Playbook Failures:** Check SSH connectivity, user permissions (`root` access required), and playbook syntax.

## Contact

For issues not covered in this documentation, please contact [Contact Information].
