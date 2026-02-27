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

1.  **Target Version Determination:** By default, the playbook determines the target K3s version from the `stable` channel (`https://update.k3s.io/v1-release/channels/stable`) using JSON response data (`tag_name`). You can pin a specific version with `-e k3s_target_version=vX.Y.Z+k3sN`.
2.  **Node Version Check:** It checks the currently installed K3s version on each node (`/usr/local/bin/k3s --version`).
3.  **Conditional Upgrade:** The upgrade steps are only performed if the node's current version does not match the target version.
4.  **Worker Node Upgrade Order:** Worker nodes are upgraded first, one by one.
    *   **Drain:** The worker node is drained using `kubectl drain` executed on the controller node via `pct exec`. The `--disable-eviction` flag is used to bypass PodDisruptionBudgets (Note: This was necessary for Vault pods and may cause temporary service disruption if PDBs are strictly required).
    *   **Upgrade:** The standard K3s upgrade script (`curl | sh`) is executed on the worker node via `pct exec`.
    *   **Uncordon:** The worker node is uncordoned using `kubectl uncordon` executed on the controller node via `pct exec`.
5.  **Controller Node Upgrade:** The controller node is upgraded last by executing the K3s upgrade script directly via `pct exec`. Draining is skipped for the single controller setup.
6.  **Post-Upgrade Smoke Check:** Run `k3s/playbook-postcheck-k3s-upgrade.yaml` to assert all nodes are `Ready` and no pods are stuck outside `Running/Completed`.

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

### Traefik Helm installer ownership conflict

If `helm-install-traefik-crd` or `helm-install-traefik` crashloops with `invalid ownership metadata`, reconcile CRD/IngressClass Helm metadata with:

```bash
ansible-playbook k3s/playbook-reconcile-traefik-crd-ownership.yaml
```

If you also have a legacy Traefik release in namespace `traefik`, avoid running two Traefik load balancers at once. Keep one as source of truth (recommended: k3s-managed `kube-system/traefik`), then scale/convert the legacy one.

## Contact

For issues not covered in this documentation, please contact [Contact Information].

## HA Migration Workflow (Terraform + Ansible)

For HA rollout, keep responsibilities split:
- **Terraform**: Provision and size controller/worker LXCs.
- **Ansible**: Configure and validate K3s control-plane lifecycle.

Recommended Ansible sequence:

1. Bootstrap/convert primary controller to embedded etcd (idempotent):

```bash
ansible-playbook k3s/playbook-bootstrap-k3s-ha.yaml
```

2. Join additional controllers as server nodes:

```bash
ansible-playbook k3s/playbook-join-k3s-ha-servers.yaml
```

3. Validate HA state and optionally create etcd snapshot:

```bash
ansible-playbook k3s/playbook-validate-k3s-ha.yaml
# optional snapshot
ansible-playbook k3s/playbook-validate-k3s-ha.yaml -e k3s_ha_take_snapshot=true
```

4. Enforce controller scheduling policy (NoSchedule taint):

```bash
ansible-playbook k3s/playbook-enforce-k3s-controller-noschedule-taint.yaml
```

5. Run routine cluster health gate (anytime):

```bash
ansible-playbook k3s/playbook-check-k3s-cluster-health.yaml
```

Notes:
- Controller taints prevent new regular workloads from landing on control-plane nodes.
- Existing pods already running on controllers will not be evicted automatically; restart/rollout those workloads to reschedule onto workers.
- If replacing the inventory-first controller (for example `k3sc1`), run join with `-e k3s_primary_controller_host=<surviving_controller>`.
- `playbook-join-k3s-ha-servers.yaml` now auto-prepares `/dev/kmsg` (conf-kmsg service) and removes stale etcd members for nodes that require rejoin.
- Join and upgrade playbooks include `pct status` preflight checks; join playbooks auto-start stopped target containers before `pct exec`.

- If no API load balancer/VIP exists yet, use the primary controller URL for initial join.
- DNS round-robin works as temporary lab mode but is not health-aware failover.

## Major LXC/OS Upgrade Canary Workflow (Workers)

When introducing a new major Rocky/template generation, use worker canary replacement instead of in-place full-cluster upgrades.

Recommended sequence:

1. Provision one extra worker LXC from the new template via Terraform.
2. Add the new worker to your Ansible inventory (`[k3s_workers]` with `pct_id` + `node_name`).
3. Ensure worker prerequisites are present (installs `nfs-utils`, configures `/dev/kmsg` helper):

```bash
ansible-playbook k3s/playbook-ensure-worker-prereqs.yaml
```

4. Join only that worker to the cluster:

```bash
ansible-playbook k3s/playbook-join-k3s-worker.yaml \
  -e k3s_join_worker_host=<new_worker_inventory_name> \
  -e k3s_join_server_url=https://k3s-api.<domain>:6443
```

5. Shift workload from old worker to new worker:

```bash
ansible-playbook k3s/playbook-canary-worker-cutover.yaml \
  -e k3s_canary_old_worker_host=<old_worker_inventory_name> \
  -e k3s_canary_new_worker_host=<new_worker_inventory_name>
```

6. Optional hard cutover cleanup:

```bash
ansible-playbook k3s/playbook-canary-worker-cutover.yaml \
  -e k3s_canary_old_worker_host=<old_worker_inventory_name> \
  -e k3s_canary_new_worker_host=<new_worker_inventory_name> \
  -e k3s_canary_stop_old_agent=true \
  -e k3s_canary_delete_old_node=true
```

7. Run cluster health checks:

```bash
ansible-playbook k3s/playbook-postcheck-k3s-upgrade.yaml
```

After validation, repeat this worker replacement flow one node at a time.

## Major Upgrade Orchestration (Repeatable As-Code)

For a reusable major upgrade flow (Rocky 10 now, Rocky 11 later), use:

- Runbook: `k3s/docs/major-upgrade-runbook.md`
- Orchestrator: `k3s/playbook-k3s-major-upgrade.yaml`
- Profile vars: `k3s/vars/major-upgrade/rocky10.yaml`

Example:

```bash
ansible-playbook k3s/playbook-k3s-major-upgrade.yaml \
  -e @k3s/vars/major-upgrade/rocky10.yaml \
  -e k3s_upgrade_canary_old_worker_host=<old_worker> \
  -e k3s_upgrade_canary_new_worker_host=<new_worker>
```

This keeps execution order explicit and reusable across future major OS generations.
