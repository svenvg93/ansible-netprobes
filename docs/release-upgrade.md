# Release upgrade

`playbooks/release-upgrade.yml` upgrades netprobes to the next Ubuntu **LTS** release (for example
24.04 → 26.04) with Ubuntu's own `do-release-upgrade`, one probe at a time. The work is done by
`roles/release_upgrade`.

## When to run it

Only when you decide to move probes to a new LTS release. Ubuntu offers an LTS-to-LTS upgrade
once the new release's first point release (`.1`) is out. It never runs as part of `site.yml` or
maintenance.

| Step | Command |
|---|---|
| 1. See what would happen | `ansible-playbook playbooks/release-upgrade.yml --limit <probe> --check` |
| 2. Upgrade one probe | `ansible-playbook playbooks/release-upgrade.yml --limit <probe>` |
| 3. Check that probe (measurements, Beszel, Tailscale) | — |
| 4. Upgrade the rest, a few at a time | `ansible-playbook playbooks/release-upgrade.yml --limit <probe>,<probe>,...` |
| Repos still switched off after an upgrade | see [Repairing the repos only](#repairing-the-repos-only) |

The playbook refuses to run without `--limit`, so the whole fleet can never be upgraded by
accident. Probes are always done one after another, and the run stops at the first failure.

Expect each probe to be offline for a while (typically 15–45 minutes), including two reboots.

## What a run does, per probe

**Checks** (also in `--check` mode):
1. Makes sure only LTS releases are offered (`Prompt=lts` in `/etc/update-manager/release-upgrades`).
2. Asks `do-release-upgrade -c` whether a new release is available. If not, the probe is skipped.
3. Checks there's enough free space on `/` and no half-installed packages (`dpkg --audit`).
4. Reports "from → to". In `--check` mode it stops here.

**Upgrade:**
5. Pauses the scheduled apt timers, so unattended-upgrades doesn't start mid-upgrade. They come
   back at the next boot.
6. **Prepare:** installs all pending updates, Tailscale included. `do-release-upgrade` refuses to
   start otherwise.
7. Reboots if the updates need it. `do-release-upgrade` also refuses to start with a reboot pending.
8. **Upgrade:** runs `do-release-upgrade` non-interactively, so every question it would ask gets
   its default answer.
9. Reboots into the new release and checks that it's running.

**Afterwards:**
10. **Turns third-party repos back on.** The upgrader switches off every non-Ubuntu repo (Tailscale,
    Docker, Grafana, InfluxData, …). `.list` files it can't convert end up as `<name>.list.disabled`
    with every line commented out; `.sources` files get `Enabled: no`. The playbook restores both,
    removes the upgrader's leftovers (`.disabled`, `.migrate`), and points release-specific repos at
    the new release name. Repos on a fixed suite such as `stable` stay as they are.
11. Refreshes the package lists.
12. Checks that `tailscaled`, `beszel-agent` and `telegraf` are running (each only if installed).

## Repairing the repos only

For a probe that's already on the new release but still has its third-party repos switched off.
You can spot that by `*.list.disabled` files (or `Enabled: no` lines) in `/etc/apt/sources.list.d`.
That happens when the upgrade ran with an older version of this playbook, or when step 10 failed.

```bash
ansible-playbook playbooks/release-upgrade.yml --limit <probe> \
  -e release_upgrade_repair_repos=true -e release_upgrade_from=<old codename>
```

- `release_upgrade_repair_repos=true` runs only step 10 (restore the repos) and step 11 (refresh
  the package lists), then stops. No upgrade, no reboot.
- `release_upgrade_from` is the release the probe was upgraded **from**, so release-specific repos
  can be moved off it. For example `noble` for a probe that came from 24.04.

It's safe to run more than once: repos that are already restored are left as they are. Add
`--check --diff` to see the changes first.

## Why it runs in the background

The probes are managed over **Tailscale SSH**: `tailscaled` carries Ansible's connection. Steps 6
and 8 restart many services, `tailscaled` included. If the upgrade ran inside the SSH session,
that restart would kill it halfway and leave the probe broken.

So both steps run as a separate systemd job (`systemd-run`) that doesn't depend on the session.
Ansible starts the job, then polls a status file every 30 seconds until the job reports it's done.
If the connection drops in between, Ansible reconnects (`ansible_ssh_retries` in the playbook) and
carries on polling.

The upgrade also replaces Python (e.g. 3.12 → 3.14). So the playbook always uses
`/usr/bin/python3`, which keeps pointing at the current version, and polls with `raw`, which
needs no Python at all.

## Logs and status

On the probe, in `/var/lib/ansible-release-upgrade/`:

| File | Content |
|---|---|
| `prepare.log`, `upgrade.log` | Full output of each step |
| `prepare.status`, `upgrade.status` | `running`, or `done <exit code>` (`done 0` is success) |
| `plan.json` | Release upgraded from and to; used to resume an interrupted run |
| `finished` | Written once all steps are done |

Ubuntu's own upgrade logs are in `/var/log/dist-upgrade/`. If a step fails, the playbook prints
the last 40 lines of its log.

## Settings

Defaults are in `roles/release_upgrade/defaults/main.yml`; override them with `-e` if needed.

| Setting | Meaning |
|---|---|
| `release_upgrade_prompt` | Which releases to offer; keep `lts` |
| `release_upgrade_min_free_gb` | Free space on `/` needed before starting |
| `release_upgrade_timeout_minutes` | How long each step may take before Ansible stops waiting |
| `release_upgrade_reboot_timeout` | Seconds to wait for a probe to come back after a reboot |
| `release_upgrade_reenable_repos` | Turn third-party repos back on afterwards |
| `release_upgrade_check_services` | Services that must be running afterwards |
| `release_upgrade_repair_repos` | Only restore the repos of an upgraded probe (see above) |
| `release_upgrade_from` | Old release codename; needed with `release_upgrade_repair_repos`, or to resume a run without `plan.json` |

## If something goes wrong

- **The run was interrupted or timed out after the upgrade:** if `upgrade.status` says `done 0`,
  just run the playbook again for that probe. It sees the upgrade succeeded and resumes at the
  reboot, then does the remaining steps. If the probe has no `plan.json` (runs from an older
  version of the playbook), add `-e release_upgrade_from=<old codename>`, e.g. `noble`.
- **An upgraded probe still has its third-party repos switched off:** see
  [Repairing the repos only](#repairing-the-repos-only).
- **"Refresh the package lists" fails with `NO_PUBKEY` after the upgrade:** a stale repo file
  without a `Signed-By` key, whose key used to come from the old global apt keyring. Check
  `apt-get update` on the probe. If the file duplicates a working repo (on netprobe11 it was
  `repos_influxdata_com_debian.sources` next to `influxdata.list`), delete it, then run the
  [repair](#repairing-the-repos-only).
- **The playbook timed out while the job was still running:** check `upgrade.status` and the log
  on the probe. Don't start a second run until it says `done`.
- **The upgrade step failed:** read `upgrade.log` and `/var/log/dist-upgrade/main.log`. Fix the
  cause, then run the playbook again for that probe. It picks up from the current state.
- **The probe doesn't come back after the reboot:** it needs hands-on access (console or physical).
  This is why you should always do one probe first.
