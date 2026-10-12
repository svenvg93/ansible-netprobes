# Maintenance

`playbooks/maintenance.yml` keeps the netprobes and the controller patched: it upgrades
packages, cleans up, reboots when Ubuntu asks for it, and reports problems. The work is done by
`roles/maintenance`.

## When to run it

| Situation | Command |
|---|---|
| Routine patching (e.g. weekly or monthly) | `ansible-playbook playbooks/maintenance.yml --limit netprobes` |
| Routine patching, controller included | `ansible-playbook playbooks/maintenance.yml -K` |
| Patch now, reboot later | `ansible-playbook playbooks/maintenance.yml --limit netprobes -e maintenance_reboot_if_required=false` |
| Careful run, one probe at a time | `ansible-playbook playbooks/maintenance.yml --limit netprobes -e maintenance_serial=1` |
| Try it on one probe first | `ansible-playbook playbooks/maintenance.yml --limit <probe>` |
| See what would change | add `--check` to any of the above |

Probes are reached as `root`, so they need no password. Only the controller needs `-K`, for its
sudo password.

## Two plays

| Play | Hosts | How |
|---|---|---|
| Maintenance (netprobes) | `netprobes` | In batches of a quarter of the probes (`serial: 25%`), rebooted when needed |
| Maintenance (controller) | `controller` | Runs last and is never rebooted by Ansible |

**Why batches:** while one batch upgrades and reboots, the other probes keep measuring, so
there's never a gap across the whole fleet. A bad update also shows up on a few probes before it
reaches all of them. Ansible only stops the whole run if *every* host in a batch fails, though.

**Why the controller runs last:** it hosts the Beszel hub in Docker, and a Docker upgrade restarts
the hub. Running it after the probes keeps that restart out of the way. Ansible can't reboot the
machine it runs on, so the role always skips the reboot there. When the report shows
`reboot_required: true` for the controller, reboot it yourself.

## What a run does, per host

1. **Hold packages** listed in `maintenance_hold_packages`. These holds are permanent;
   release one with `sudo apt-mark unhold <pkg>`.
2. **Upgrade packages** with apt (`update`, then the upgrade, `autoremove` and `autoclean`).
   - **Tailscale is skipped.** It's held for the duration of the upgrade and released afterwards,
     even if the upgrade fails (see below).
   - apt waits up to 5 minutes for the apt lock, so it doesn't fail when unattended-upgrades
     is running.
3. **Vacuum the journal:** delete systemd journal logs older than `maintenance_journal_max_age`.
4. **Reboot** if Ubuntu flagged that one is needed (`/var/run/reboot-required`) and
   `maintenance_reboot_if_required` is on. Ansible waits up to 10 minutes for the host to come back.
5. **Report** per host: whether packages changed and whether a reboot was needed or done.
   Pending reboots are also shown as a warning.
6. **Check the Beszel agent:** fail the host if the agent is installed but not running.

## Why Tailscale is held during the upgrade

The probes are managed over **Tailscale SSH**: `tailscaled` carries the connection Ansible is
using. Upgrading the `tailscale` package restarts `tailscaled`, which cuts that connection
partway through the apt run and can leave packages half-configured.

So maintenance never upgrades Tailscale. Tailscale's own **auto-update** (switched on by
bootstrap) installs new versions instead, in a way that doesn't depend on an SSH session.

The hold is **temporary**. On Linux, Tailscale's updater uses apt, so a permanent hold would
block it too. Tailscale also releases new versions gradually over several days, so a probe
running one version behind the apt repo is normal.

## Settings

In `inventory/group_vars/netprobes/main.yml`. Defaults are in `roles/maintenance/defaults/main.yml`.

| Setting | Meaning |
|---|---|
| `maintenance_upgrade` | apt upgrade mode: `dist`, `safe`, `full` or `yes` |
| `maintenance_reboot_if_required` | Reboot when Ubuntu asks for it (always off for the controller) |
| `maintenance_journal_max_age` | Journal logs older than this are deleted |
| `maintenance_hold_packages` | Packages never upgraded (permanent hold) |
| `maintenance_verify_beszel` | Fail a host whose Beszel agent isn't running |
| `maintenance_skip_upgrade_packages` | Held only during the upgrade (default: `tailscale`) |
| `maintenance_serial` | Batch size for the probes; set it with `-e` |
