# Proxmox k3s Cluster Management

This repository contains the Ansible playbooks and configurations for managing a lightweight Kubernetes (k3s) cluster hosted on Proxmox Virtual Environment (PVE) using Linux Containers (LXC). The setup automates the deployment, management, and upgrading of k3s clusters, providing a streamlined and consistent approach to Kubernetes cluster management.

## Project Structure

- **`/inventory`**: Contains Ansible inventory files, including hosts and group variables, that define the cluster nodes and their roles.
- **`/k3s`**: Houses playbooks specific to k3s operations such as installation, upgrade, and node management.
  - `playbook-upgrade-k3s.yaml`: An Ansible playbook for upgrading k3s versions across the cluster.
  - `playbook-check-k3s-version.yaml`: A playbook for checking the installed k3s version on all nodes.
  - `playbook-postcheck-k3s-upgrade.yaml`: A post-upgrade smoke check (node readiness + pod health).
  - `playbook-refresh-k3s-kubeconfig.yaml`: Refreshes local kubeconfig from the controller (and rotates certs if expired).
  - **`K3S.md`**: Detailed documentation on the k3s setup, architecture, and management practices.
- **`ansible.cfg`**: Ansible configuration file that specifies default settings for the project, including the inventory path.
- **`README.md`**: Provides an overview of the repository, setup instructions, and how to contribute.

## Getting Started

To manage your k3s cluster, you'll need to set up your environment with Ansible and configure SSH access to your Proxmox server and LXC containers. Follow these steps:

1. **Install Ansible**: Ensure Ansible is installed on your machine. For installation instructions, refer to the [official documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).
2. **Configure SSH Access**: Set up SSH keys for passwordless access to your Proxmox server. See [SSH key management](https://www.ssh.com/academy/ssh/keygen) for more details.
3. **Set Up Inventory**: Populate the `/inventory/hosts.ini` file with your Proxmox and k3s node details. Refer to `./inventory/notes.md` for guidance on the inventory structure.
4. **Run Playbooks**: Execute Ansible playbooks to manage your cluster. For example, to check k3s versions on all nodes, run:
   ```bash
   ansible-playbook k3s/playbook-check-k3s-version.yaml
   ```

## Setting Up SSH Access

Before running the playbooks, you'll need to set up SSH access to your Proxmox server. An example SSH configuration template (`ssh_config.example`) is provided in this repository. To use it:

1. Copy `ssh_config.example` to your SSH configuration folder, usually `~/.ssh/config`, and rename it appropriately.
2. Replace `<proxmox_ip_or_hostname>`, `<user>`, and `<path_to_your_ssh_private_key>` with your actual data.
3. Ensure your private key permissions are correctly set by running `chmod 600 <path_to_your_ssh_private_key>`.

This setup will allow Ansible to communicate securely with your Proxmox server.
