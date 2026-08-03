# The Phase System

Design documentation for the resumable upgrade demo. This is a distilled,
runnable version of the checkpoint system used in production for DWDM optical
and backbone router upgrades.

## Why

Device upgrades are long, stateful, and disruptive. When automation fails
mid-run, restarting from scratch risks re-running disruptive operations
(traffic drain, software install) against a device that is already partially
upgraded. The playbook needs memory of where it was.

## Phase Model

The upgrade is a linear sequence of named phases with numeric ordering,
defined once in [vars/phase_order.yaml](../vars/phase_order.yaml):

```
validation (10) → pre_maint (20) → traffic_remove (30)
→ upgrade_install (40) → reload_wait (50) → upgrade_verify (60)
→ traffic_restore (70) → post_maint (80) → cleanup (90)
       |
(sub-phases within traffic_remove)
traffic_remove_drain (31) → traffic_remove_verify (32)
```

Every task in the workflow is gated with the same conditional:

```yaml
when: current_phase_order | int <= phase_order['<phase>'] | int
```

`current_phase_order` is set once at playbook start. Any phase whose order is
below it is skipped — that is the entire skipping mechanism.

## Checkpoints

After each phase **succeeds**, a YAML checkpoint is written to
`.checkpoints/<host>_<ticket>.yaml`
([tasks/shared/checkpoint_save.yaml](../tasks/shared/checkpoint_save.yaml)):

```yaml
phase: traffic_remove          # major phase last completed
sub_phase: traffic_remove_drain  # more precise sub-phase, when applicable
ticket: DEMO-0001              # checkpoint is scoped per device + ticket
inventory_hostname: router-a.demo.net
target_version: 24.3-DEMO
protocol_baseline:             # facts later phases depend on
  isis_adjacencies: 4
  bgp_peers: 12
timestamp: '2026-08-01 10:15:00'
```

**Checkpoint-on-complete, not checkpoint-on-start**: a phase that fails midway
leaves the previous checkpoint in place, so resume re-runs the failed phase
from its beginning.

**Persist every phase-gating fact**: the checkpoint carries not just the phase
pointer but any fact a later phase depends on (here, the protocol baseline
captured in `pre_maint` and compared in `upgrade_verify`). A resumed run must
be able to reconstruct the original run's context entirely from the checkpoint.

## Resume Rules

At playbook start, [tasks/shared/checkpoint_load.yaml](../tasks/shared/checkpoint_load.yaml)
decides where the run begins:

| Situation | Behavior |
|-----------|----------|
| `start_from_phase=auto`, no checkpoint | Fresh run from `validation` |
| `start_from_phase=auto`, valid checkpoint | Resume at the phase **after** the completed one |
| Checkpoint for a different ticket | Refused before any work begins |
| Checkpoint from a completed run (`cleanup`) | Refused — nothing to resume; delete it to start fresh |
| `start_from_phase=<name>` | Start **exactly at** that phase (operator override); saved facts still restored |

Auto-resume never re-runs the completed phase; manual override deliberately
can — that is how an operator forces a re-run of a specific phase.

## Non-Idempotent Operations

Most tasks can safely run twice. The software install cannot: re-issuing it
against a device that is already installing/rebooting could corrupt the
install or cause a second disruptive reboot.

Before issuing the install,
[upgrade_install.yaml](../roles/upgrade_demo/tasks/upgrade_install.yaml)
runs a **pre-flight check** (in production a NETCONF reachability probe; here
a marker file):

```
Reboot already in progress?
  yes → skip the install, advance the phase pointer to reload_wait
  no  → issue the install normally
```

This means the playbook can be safely re-launched even if it died *between*
issuing the install and saving the checkpoint — the one window where the
checkpoint alone is not enough.

## Verify Before Restore

Traffic is only restored after the convergence gate passes: the post-upgrade
protocol state must match the baseline captured before the maintenance began.
A failed gate stops the run with the device still drained — and resumable.

## Mapping to Production

| Demo | Production |
|------|------------|
| `debug` messages | Vendor modules (NETCONF/CLI) against real devices |
| `.rebooting` marker file | NETCONF port probe / install status query |
| `traffic_remove` (2 sub-phases) | 7 checkpointed drain sub-phases (IGP overload, eBGP shutdown, MPLS drain, ...) |
| `protocol_baseline` (2 counters) | ISIS/RSVP/LDP baselines, hardware baselines, ServiceNow record identity, reboot flags |
| `-e` extra vars | AWX survey variables |
| `.checkpoints/` in repo | Persistent path on the AWX controller |
