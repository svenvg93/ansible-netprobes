# AGENTS.md

Instructions for AI agents working in this repo. Read this before changing anything.
User-facing docs are in `README.md`; this file covers what an agent needs on top of that.

## What this is

Ansible for a fleet of 16 Ubuntu 24.04 "netprobes" (`netprobe01`–`netprobe16`) that run
internet measurements. This repo handles:

- **bootstrap** – timezone, package cleanup, snap removal, Tailscale auto-update, Ookla Speedtest CLI
- **beszel** – Beszel monitoring agent via the `community.beszel.agent` collection role
- **maintenance** – apt upgrades, journal cleanup, reboots, health checks

Telegraf and Grafana Alloy (which run the actual measurements) are out of scope for this repo.
Ignore `../ansible`: don't read it, copy from it or change it.

## Environment facts

- **Controller** is `cpe-srv` (this machine), in the `controller` group with `ansible_connection: local`.
  It only gets maintenance and is never rebooted by Ansible (the role skips reboots for local hosts).
  Its sudo needs a password, so runs that include it need `-K`, which you can't type. Use `--limit netprobes`.
- **Netprobes** are reached over **Tailscale SSH** as `root` (`ansible_become: false`), with the
  address `<name>.tail294b9a.ts.net`. No password and no `-K`.
- Several probes are offline in Tailscale (as of 2026-10-08: 04, 09–12, 15, 16). `UNREACHABLE` for
  those is normal. 10, 12 and 15 have been offline for months; ask before removing them from the inventory.
- The **Beszel hub** runs in Docker from `~/docker/beszel` (hub + Tailscale sidecar, TS hostname `beszel`).
  Agents connect out to `https://beszel.tail294b9a.ts.net` over WebSocket with a universal token.
- All reachable probes are x86_64 with a single NVMe drive (`/dev/nvme0`).

## How to verify changes

Always run these before saying a change works:

```bash
~/.local/bin/ansible-lint                                   # must pass at the 'production' profile
ansible-playbook playbooks/site.yml --syntax-check
ansible-playbook playbooks/<play>.yml --limit netprobe01 --check --diff   # dry run on a real probe
```

- Use the full path `~/.local/bin/ansible-lint`. `/usr/bin/ansible-lint` is an old, broken apt version.
- Fix lint findings rather than silencing them. Don't add an `.ansible-lint` config that skips rules
  or lowers the profile. A `# noqa: <rule>` is only OK when no module can do the job, and it needs a
  comment above it saying why. The one existing case: `systemctl list-units --state=failed` in
  `roles/maintenance/tasks/main.yml`, because `service_facts` misses failed timers and mounts.
- Redirect stdin (`</dev/null`) for every `ansible`/`ansible-playbook` call from a non-interactive
  shell, or Ansible fails with "requires blocking IO on stdin/stdout/stderr".
- Ad-hoc read-only checks against probes are fine, e.g. `ansible netprobes -o -m command -a '...' </dev/null`.
- **Never do a real (non-check) run against probes without the user's explicit OK.** These are
  production measurement hosts.
- Known check-mode limits (not bugs): `beszel.yml` stops at the collection's "start service" task on
  probes without the agent. The speedtest role reports "Would install" instead of downloading.
  Use `--start-at-task` to dry-run tasks after a failing one.

## Conventions

- Match the existing style: FQCN modules (`ansible.builtin.*`), role-prefixed variable names,
  short comments that explain *why*, `changed_when`/`check_mode: false` on read-only commands.
- Settings go in `inventory/group_vars/netprobes/main.yml`, defaults in each role's `defaults/main.yml`.
  The user prunes settings they don't need, so don't re-add removed ones (e.g. `agent_version`,
  `maintenance_disk_warn_percent`; the disk check was removed because Beszel covers it).
- Secrets go only in `inventory/group_vars/netprobes/vault.yml` (ansible-vault, git-ignored).
  `vault.yml.example` documents the keys.
- Keep `README.md` in sync when behaviour or commands change.
- Commit only when asked. Commits end with the `Co-Authored-By` trailer from the session instructions.
  `.claude/` is git-ignored on purpose. Remote: `github.com/svenvg93/ansible-netprobes`.

## Gotchas (read before touching these areas)

- **Tailscale + apt**: upgrading the `tailscale` package restarts `tailscaled`, which kills the
  Tailscale SSH session mid-apt. Maintenance holds it only during the upgrade
  (`maintenance_skip_upgrade_packages`) and always releases it, so Tailscale's own auto-update keeps
  working. Never hold it permanently, and never upgrade it over an Ansible session. Tailscale rolls
  out its own updates over days; a pending version is not a bug.
- **needrestart** is removed by bootstrap for the same reason: it auto-restarts services
  (including `tailscaled`/`ssh`) after apt.
- **Speedtest** is installed from Ookla's tarball (pinned version + sha256) to **`/usr/bin/speedtest`**,
  the exact path Telegraf calls. Don't move it. Removing the old apt package deletes that file, so
  the role removes the package first, then installs the binary.
- **Beszel**: `agent_public_key` in group_vars is still a placeholder, and `vault.yml` (with
  `vault_agent_token`) doesn't exist yet. `beszel.yml` asserts on the token. The user wants to add
  `DISABLE_SSH` support to `community.beszel` upstream (a PR), so **don't** add a local drop-in
  workaround. Use the new collection variable once that's released (see README, "Updating the collections").
- **Maintenance order**: `site.yml` runs bootstrap → beszel → maintenance, because maintenance fails
  hosts whose `beszel-agent` unit exists but isn't running.
- `community.beszel` installs the agent to `/usr/local/bin`. Some probes still have
  `/opt/beszel-agent` left over from the official install script.

## Open items (as of 2026-10-08)

- GitHub Actions workflow for lint/syntax checks: requested, not started.
- Push to GitHub is blocked until the user re-runs `gh auth login` (the stored token is invalid).
- Discussed but not decided: scheduled maintenance (systemd timer on `cpe-srv`), `log_path` in
  `ansible.cfg`, cleanup of `/opt/beszel-agent` leftovers, removing long-offline probes.
