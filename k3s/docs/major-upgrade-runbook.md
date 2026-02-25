# K3s Major LXC Upgrade Runbook

## Objective
Repeatable, as-code process for major OS/template node replacement (Rocky 10 now, Rocky 11 later).

## Responsibility Split
- Terraform: build template and provision replacement LXCs.
- Ansible: join/cutover/validation and optional hardware post-configuration.

## Playbook Catalog
- `k3s/playbook-k3s-upgrade-precheck.yaml`
  - Validates current cluster health and required upgrade profile inputs.
- `k3s/playbook-lxc-template-build.yaml`
  - Validates template metadata used as Terraform handoff.
- `k3s/playbook-k3s-node-replace.yaml`
  - Replaces one worker (`old -> new`) by composing join + canary cutover.
- `k3s/playbook-enable-worker-gpu-passthrough.yaml`
  - Optional hardware pass-through and node labeling for Plex workloads.
- `k3s/playbook-k3s-upgrade-postcheck.yaml`
  - Final health gate.
- `k3s/playbook-k3s-major-upgrade.yaml`
  - Canary orchestrator for one replacement cycle.

## Profile Variables
Use profile files and pass them with `-e @...`.

- `k3s/vars/major-upgrade/rocky10.yaml`
- `k3s/vars/major-upgrade/rocky11.yaml`

## Execution Order
1. Provision replacement node in Terraform from target template.
2. Add new node to inventory (`pct_id`, `node_name`, group membership).
3. Run canary orchestration:

```bash
ansible-playbook k3s/playbook-k3s-major-upgrade.yaml \
  -e @k3s/vars/major-upgrade/rocky10.yaml \
  -e k3s_upgrade_canary_old_worker_host=<old_worker> \
  -e k3s_upgrade_canary_new_worker_host=<new_worker>
```

4. If this node hosts GPU workloads, run GPU pass-through:

```bash
ansible-playbook k3s/playbook-enable-worker-gpu-passthrough.yaml \
  -e k3s_gpu_worker_host=<new_worker>
```

5. Repeat step 1-4 for each worker (`serial: 1` operationally).
6. Controllers only after workers are stable.

## Manual Guardrails
- Keep at least one extra schedulable worker capacity during each cutover.
- Do not parallelize replacements.
- Keep previous template available until postchecks pass.

## Rocky 11 Reuse
- Reuse the same playbook order and commands.
- Change only template/version/profile values.
