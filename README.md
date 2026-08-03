# Resumable Upgrade Demo[text](https://github.com/jnicholson56/ansible_phase_demo)

A minimal, runnable demonstration of the **phase-aware checkpoint system** used to
make long network device upgrades resumable after a failure.

All device operations are **simulated** (debug messages and short waits) — no
network devices are touched, and no credentials are needed. The phase gating,
checkpointing, auto-resume, pre-flight safety checks, and operator overrides
are all fully functional and behave exactly like the production system.

## How It Works (in 30 seconds)

- The upgrade is a linear sequence of named phases, each with a numeric order
  (see [vars/phase_order.yaml](vars/phase_order.yaml)).
- Every task is gated: `when: current_phase_order <= phase_order['<phase>']`.
- After each phase **succeeds**, a checkpoint file is written to `.checkpoints/`
  recording the completed phase (per device, per change ticket).
- On the next run, the checkpoint is read and the playbook resumes at the
  **next** phase after the one that completed — a failed phase is re-run from
  its beginning, and completed phases are never repeated.

See [docs/PHASE_SYSTEM.md](docs/PHASE_SYSTEM.md) for the full design.

## Layout

```
example/
├── playbooks/
│   └── device_upgrade.yaml        # entry point: validation → upgrade → cleanup
├── roles/
│   └── upgrade_demo/              # the phase sequence with simulated device work
├── tasks/
│   └── shared/
│       ├── checkpoint_load.yaml   # decide where this run starts
│       └── checkpoint_save.yaml   # record a completed phase
├── vars/
│   └── phase_order.yaml           # single source of truth for phase ordering
├── inventory/                     # two fake routers, ansible_connection: local
└── docs/
    └── PHASE_SYSTEM.md            # design documentation
```

## Requirements

- `ansible-core` 2.14 or later. Nothing else — only `ansible.builtin` modules are used.

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

## Demo Script

Run everything from this directory. Use `--limit router-a.demo.net` for
single-device output.

### 1. Fresh run (happy path)

```bash
ansible-playbook playbooks/device_upgrade.yaml
```

Every phase runs in order. Watch for the `CHECKPOINT SAVED` messages, then
inspect a checkpoint file:

```bash
cat .checkpoints/router-a.demo.net_DEMO-0001.yaml
```

### 2. Fail mid-run, then resume

Reset first (`rm -rf .checkpoints`), then inject a failure during the traffic
drain:

```bash
ansible-playbook playbooks/device_upgrade.yaml -e simulate_failure_at=traffic_remove_drain
```

The job fails, and the checkpoint shows the last phase that *completed*
(`pre_maint`). Now simply re-run with no arguments:

```bash
ansible-playbook playbooks/device_upgrade.yaml
```

`start_from_phase` defaults to `auto`: the run resumes at `traffic_remove_drain`
(the failed phase, from its beginning) and skips everything before it.

### 3. Non-idempotent operation protection (pre-flight)

Reset, then fail immediately *after* the install command was issued:

```bash
rm -rf .checkpoints
ansible-playbook playbooks/device_upgrade.yaml -e simulate_failure_at=upgrade_install
ansible-playbook playbooks/device_upgrade.yaml
```

On the re-run, the pre-flight check detects the reboot already in progress
(via the `.rebooting` marker file, which stands in for a real NETCONF probe)
and **skips the install command entirely**, jumping straight to `reload_wait`.
The install is never issued twice.

### 4. Operator override

Start exactly at a chosen phase, ignoring the auto-resume logic:

```bash
ansible-playbook playbooks/device_upgrade.yaml -e start_from_phase=traffic_restore
```

This is how an operator re-runs a specific phase deliberately — for example,
restoring traffic on a device that was upgraded manually.

### 5. Completed-run protection

After a successful end-to-end run, re-run with defaults:

```bash
ansible-playbook playbooks/device_upgrade.yaml
```

The stale checkpoint is **rejected** — the previous run completed, so there is
nothing to resume. The checkpoint remains as an audit artifact; remove it to
start over:

```bash
rm -rf .checkpoints
```

## Variables

All defaults live in [inventory/group_vars/all.yml](inventory/group_vars/all.yml)
and can be overridden with `-e` (they map to AWX survey variables in production):

| Variable | Default | Purpose |
|----------|---------|---------|
| `ticket` | `DEMO-0001` | Change ticket; checkpoints are scoped per device + ticket |
| `start_from_phase` | `auto` | `auto` = resume from checkpoint; or any phase name |
| `simulate_failure_at` | *(empty)* | Inject a failure during the named phase (demo only) |
| `target_version` | `24.3-DEMO` | Pretend software version to "install" |
