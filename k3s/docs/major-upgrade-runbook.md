# K3s Major LXC Upgrade Runbook

## Objective
Repeatable, as-code process for major OS/template node replacement (Rocky 10 now, Rocky 11 later), including Plex state protection.

## Responsibility Split
- Terraform: build template and provision replacement LXCs.
- Ansible: backup/restore app state, join/cutover/validation, and optional hardware post-configuration.

## Playbook Catalog
- `k3s/playbook-k3s-upgrade-precheck.yaml`
  - Validates current cluster health and required upgrade profile inputs.
- `k3s/playbook-lxc-template-build.yaml`
  - Validates template metadata used as Terraform handoff.
- `k3s/playbook-backup-plex-config.yaml`
  - Creates a consistent backup of `/volumes/plex-config` in the selected worker LXC.
- `k3s/playbook-k3s-node-replace.yaml`
  - Replaces one worker (`old -> new`) by composing join + canary cutover.
- `k3s/playbook-enable-worker-gpu-passthrough.yaml`
  - Optional hardware pass-through and node labeling for Plex workloads.
- `k3s/playbook-restore-plex-config-from-vzdump.yaml`
  - Restores Plex config from a Proxmox `vzdump` archive into a target worker.
- `k3s/playbook-k3s-upgrade-postcheck.yaml`
  - Final health gate.
- `k3s/playbook-k3s-major-upgrade.yaml`
  - Canary orchestrator for one replacement cycle.

## Profile Variables
Use profile files and pass them with `-e @...`.

- `k3s/vars/major-upgrade/rocky10.yaml`
- `k3s/vars/major-upgrade/rocky11.yaml`

## Execution Order
1. Run prechecks.
2. If worker hosts Plex state, capture Plex backup before replacement.
3. Provision replacement node in Terraform from target template.
4. Add new node to inventory (`pct_id`, `node_name`, group membership).
5. Run canary orchestration (`old -> new`).
6. If this node hosts GPU workloads, run GPU pass-through and label workflow.
7. If Plex claim/library was lost, restore from `vzdump` archive.
8. Run postchecks.
9. Repeat for next worker (`serial: 1` operationally).
10. Controllers only after workers are stable.

## Commands

### 1) Backup Plex config
```bash
ansible-playbook k3s/playbook-backup-plex-config.yaml \
  -e k3s_plex_worker_host=<worker_host>
```

### 2) Worker replacement (canary)
```bash
ansible-playbook k3s/playbook-k3s-major-upgrade.yaml \
  -e @k3s/vars/major-upgrade/rocky10.yaml \
  -e k3s_upgrade_canary_old_worker_host=<old_worker> \
  -e k3s_upgrade_canary_new_worker_host=<new_worker>
```

### 3) Optional GPU post-step
```bash
ansible-playbook k3s/playbook-enable-worker-gpu-passthrough.yaml \
  -e k3s_gpu_worker_host=<new_worker>
```

### 4) Restore Plex config from Proxmox backup
```bash
ansible-playbook k3s/playbook-restore-plex-config-from-vzdump.yaml \
  -e k3s_plex_worker_host=<worker_host> \
  -e k3s_plex_restore_archive=/mnt/pve/diskstation/dump/vzdump-lxc-109-2026_02_22-01_08_02.tar.zst
```

## Manual Guardrails
- Keep at least one extra schedulable worker capacity during each cutover.
- Do not parallelize replacements.
- Keep previous template available until postchecks pass.
- Keep at least one recent Plex backup/restore point before each worker migration.

## Rocky 11 Reuse
- Reuse the same playbook order and commands.
- Change only template/version/profile values.
