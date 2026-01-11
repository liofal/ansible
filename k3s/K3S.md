# K3S Cluster Management Documentation

## Overview

This document outlines the structure and management practices of our Kubernetes (k3s) cluster, hosted on Proxmox Virtual Environment (PVE) using Linux Containers (LXC). The cluster consists of both controller and worker nodes managed through Ansible automation for tasks such as upgrades, configuration changes, and routine maintenance.

## Architecture

- **Proxmox VE (PVE)**: Hosts our LXC containers running k3s components.
- **K3s Cluster**: A lightweight Kubernetes cluster comprising:
  - **Controller Node(s)**: Runs the Kubernetes control plane.
  - **Worker Node(s)**: Hosts the deployed applications and services.

## Installation

K3s installation within LXC containers is managed through Ansible. The installation script is fetched directly from k3s.io and executed within each container, specifying the node role (controller or worker) and other configurations.

## Upgrading K3S

Upgrades are performed using the `k3s/playbook-upgrade-k3s.yaml` Ansible playbook. The process is designed to be idempotent and minimize disruption:

1.  **Target Version Determination:** The playbook automatically determines the latest stable K3s version by querying the `https://update.k3s.io/v1-release/channels/stable` endpoint.
2.  **Node Version Check:** It checks the currently installed K3s version on each node (`/usr/local/bin/k3s --version`).
3.  **Conditional Upgrade:** The upgrade steps are only performed if the node's current version does not match the target version.
4.  **Worker Node Upgrade Order:** Worker nodes are upgraded first, one by one.
    *   **Drain:** The worker node is drained using `kubectl drain` executed on the controller node via `pct exec`. The `--disable-eviction` flag is used to bypass PodDisruptionBudgets (Note: This was necessary for Vault pods and may cause temporary service disruption if PDBs are strictly required).
    *   **Upgrade:** The standard K3s upgrade script (`curl | sh`) is executed on the worker node via `pct exec`.
    *   **Uncordon:** The worker node is uncordoned using `kubectl uncordon` executed on the controller node via `pct exec`.
5.  **Controller Node Upgrade:** The controller node is upgraded last by executing the K3s upgrade script directly via `pct exec`. Draining is skipped for the single controller setup.

This ensures worker nodes are gracefully handled, and upgrades only occur when necessary.

## Inventory Structure

Refer to `./inventory/notes.md` for detailed information on the inventory structure and how nodes are organized for Ansible automation.

## Maintenance Procedures

Routine maintenance tasks, including backup and monitoring, are outlined here. [Details on specific procedures, backup schedules, monitoring tools.]

## Troubleshooting

Common issues and their resolutions are documented here to aid in rapid problem identification and resolution. [Include troubleshooting steps for frequent issues encountered in the environment.]

### kubectl auth failure (expired client certificate)

If `kubectl` fails with errors like `x509: certificate has expired or is not yet valid`, refresh your local kubeconfig from the controller:

```bash
ansible-playbook k3s/playbook-refresh-k3s-kubeconfig.yaml
```

Notes:
- The playbook reads `/etc/rancher/k3s/k3s.yaml` from the controller LXC via `pct exec` on the Proxmox host, rewrites the `server:` field to use the controller `node_name`, and writes the result to your first `KUBECONFIG` path (or `~/.kube/k3s-<controller>.config` if `KUBECONFIG` is unset).
- If the controller's kubeconfig client certificate is already expired, the playbook will stop `k3s`, run `k3s certificate rotate`, and start `k3s` again before re-fetching the kubeconfig. Disable this behavior with `-e k3s_kubeconfig_rotate_if_expired=false`.

## Contact

For issues not covered in this documentation, please contact [Contact Information].
