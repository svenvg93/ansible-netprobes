# Reboot

`playbooks/reboot.yml` reboots netprobes on demand, a few at a time, and checks that each one
comes back healthy. It never touches the controller and isn't part of `site.yml`.

For reboots after patching you don't need it: maintenance already reboots when Ubuntu asks for it.
Use this playbook when you want a reboot without patching, e.g. after a manual change or to clear
a stuck probe.

## When to run it

| Situation | Command |
|---|---|
| Reboot one or a few probes | `ansible-playbook playbooks/reboot.yml --limit <probe>[,<probe>...]` |
| Reboot the whole fleet | `ansible-playbook playbooks/reboot.yml` |
| Only probes with a pending reboot | `ansible-playbook playbooks/reboot.yml -e reboot_only_if_required=true` |
| One probe at a time | `ansible-playbook playbooks/reboot.yml -e reboot_serial=1` |
| See which probes would reboot | add `--check` to any of the above |

## What a run does, per probe

1. Checks whether Ubuntu has flagged a reboot (`/var/run/reboot-required`).
2. **Reboots**, unless `reboot_only_if_required` is on and no reboot is pending. Ansible waits up
   to 10 minutes for the probe to come back over Tailscale SSH.
3. **Checks the Beszel agent:** fails the probe if the agent is installed but not running.

Probes are rebooted in batches of a quarter of the fleet (`serial: 25%`), so the others keep
measuring. Unlike maintenance, the run **stops at the first probe that fails** (doesn't come back,
or the Beszel check fails), before the next batch is rebooted.

## Settings

Set them with `-e`.

| Setting | Default | Meaning |
|---|---|---|
| `reboot_serial` | `25%` | Batch size |
| `reboot_only_if_required` | `false` | Only reboot probes where Ubuntu asks for it |
| `reboot_timeout` | `600` | Seconds to wait for a probe to come back |
