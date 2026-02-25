# Ansible Inventory Structure for K3S Cluster Management

## Overview

This document provides details on the Ansible inventory structure used for managing our k3s cluster within Proxmox LXC containers.

## Inventory Groups

- **`[proxmox_server]`**: Lists the Proxmox host(s) where LXC containers are running.
- **`[k3s_controller]`**: Contains the controller node(s) of the k3s cluster.
  - For HA, include three controllers in this group.
  - The first entry is treated as the primary bootstrap controller by HA playbooks.
- **`[k3s_workers]`**: Includes all worker nodes part of the k3s cluster.
- **`[rockylinux_containers]`** (optional): Convenience group for OS update playbooks.

## Host Variables

Each host in the inventory is associated with specific variables:

- **`pct_id`**: The ID of the LXC container on the Proxmox server.
- **`node_name`**: The Kubernetes node name, used for management via `kubectl`.

Example:

```ini
[k3s_controller]
k3sc1 pct_id=103 node_name=k3sc1.liofal.net
k3sc2 pct_id=108 node_name=k3sc2.liofal.net
k3sc3 pct_id=107 node_name=k3sc3.liofal.net

[k3s_workers]
k3sw1 pct_id=102 node_name=k3sw1.liofal.net
k3sw2 pct_id=109 node_name=k3sw2.liofal.net
```
