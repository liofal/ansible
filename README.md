# Proxmox k3s Cluster Management

This repository contains the Ansible playbooks and configurations for managing a lightweight Kubernetes (k3s) cluster hosted on Proxmox Virtual Environment (PVE) using Linux Containers (LXC). The setup automates the deployment, management, and upgrading of k3s clusters, providing a streamlined and consistent approach to Kubernetes cluster management.

## Project Structure

- **`/inventory`**: Contains Ansible inventory files, including hosts and group variables, that define the cluster nodes and their roles.
- **`/k3s`**: Houses playbooks specific to k3s operations such as installation, upgrade, and node management.
  - `playbook-upgrade-k3s.yaml`: An Ansible playbook for upgrading k3s versions across the cluster.
  - `check-k3s-version.yaml`: A playbook for checking the installed k3s version on all nodes.
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
   ansible-playbook k3s/check-k3s-version.yaml

## Setting Up SSH Access

Before running the playbooks, you'll need to set up SSH access to your Proxmox server. An example SSH configuration template (`ssh_config.example`) is provided in this repository. To use it:

1. Copy `ssh_config.example` to your SSH configuration folder, usually `~/.ssh/config`, and rename it appropriately.
2. Replace `<proxmox_ip_or_hostname>`, `<user>`, and `<path_to_your_ssh_private_key>` with your actual data.
3. Ensure your private key permissions are correctly set by running `chmod 600 <path_to_your_ssh_private_key>`.

This setup will allow Ansible to communicate securely with your Proxmox server.

# Configuring SSH for Proxmox Access

To securely manage your Proxmox servers via SSH, follow these steps:

1. **Prepare Your SSH Key**: If you haven't already, generate an SSH key pair using `ssh-keygen` or ensure you have an existing key that you want to use.

2. **Configure SSH**: Use the `ssh_config.example` as a template for creating your SSH configuration:
    - Copy `ssh_config.example` to `~/.ssh/config`.
    - Replace placeholders with your actual data:
        - `<proxmox_ip_or_hostname>` with your Proxmox server's IP address or hostname.
        - `<user>` with the username you'll use for SSH connections, typically `root` for Proxmox.
        - `<path_to_your_ssh_private_key>` with the full path to your private SSH key.
    - Adjust file permissions with `chmod 600 <path_to_your_ssh_private_key>` to secure your private key.

3. **Test SSH Connection**: Ensure you can connect to your Proxmox server using SSH without needing to enter a password.

Remember, do not commit actual configuration files containing sensitive information to public repositories. Always use and share template or example files with placeholders.
