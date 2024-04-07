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

Upgrades are performed via Ansible playbooks that execute the k3s upgrade script within each LXC container. The upgrade process includes draining Kubernetes nodes, updating k3s, and then uncordoning the nodes to re-enable pod scheduling. This ensures minimal disruption to running services.

## Inventory Structure

Refer to `./inventory/notes.md` for detailed information on the inventory structure and how nodes are organized for Ansible automation.

## Maintenance Procedures

Routine maintenance tasks, including backup and monitoring, are outlined here. [Details on specific procedures, backup schedules, monitoring tools.]

## Troubleshooting

Common issues and their resolutions are documented here to aid in rapid problem identification and resolution. [Include troubleshooting steps for frequent issues encountered in the environment.]

## Contact

For issues not covered in this documentation, please contact [Contact Information].
