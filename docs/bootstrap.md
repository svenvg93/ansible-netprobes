# Bootstrap

`playbooks/bootstrap.yml` brings a netprobe to the standard setup. It runs two roles:
`bootstrap` (system baseline) and `speedtest` (Ookla Speedtest CLI). Every step checks the
current state first, so running it again on an existing probe only fixes what has drifted.

## When to run it

| Situation | Command |
|---|---|
| New or rebuilt probe | `ansible-playbook playbooks/bootstrap.yml --limit <probe>` |
| After changing a bootstrap setting | `ansible-playbook playbooks/bootstrap.yml` |
| Fix one part only | `ansible-playbook playbooks/bootstrap.yml --tags <tag>` |
| After changing the speedtest version | `ansible-playbook playbooks/bootstrap.yml --tags speedtest` |
| See what would change | add `--check --diff` to any of the above |

Tags: `timezone`, `packages`, `tailscale`, `ubuntu_pro`, `speedtest`.

Probes that are offline in Tailscale are skipped with a note; run it for them once they're back.

For a completely new probe, `playbooks/site.yml` runs bootstrap, then the Beszel agent, then
maintenance.

## Timezone (`timezone`)

Sets the system timezone to `bootstrap_timezone`.

## Packages (`packages`)

- **Installs** the baseline packages in `bootstrap_install_packages`.
- **Removes and purges** the packages in `bootstrap_remove_packages`, then autoremoves leftovers:

| Package | Why it goes |
|---|---|
| `lxd-installer` | Installs LXD from snap on first use; not needed on probes |
| `modemmanager` | Probes don't use mobile modems; it also probes serial devices |
| `landscape-common` | Canonical Landscape client and login message; not used |
| `popularity-contest` | Sends package usage statistics to Ubuntu |
| `needrestart` | Restarts services automatically after apt, including `tailscaled` and `ssh`, which cuts the Ansible session mid-run |
| `snapd` | Snap isn't used on the probes. Always removed, with its snaps first, and pinned so it can't come back (see below) |

### Snap removal

snapd can't simply be purged like the packages above: its snaps have to be removed first, in
order. This runs as part of the `packages` step, only if snapd is installed:

1. Removes application snaps, then base snaps (`core*`, `bare`), then the `snapd` snap itself.
   Bases go last because apps depend on them.
2. Purges the `snapd` apt package.

Then, on every run:

3. **Pins snapd** at priority -10 in `/etc/apt/preferences.d/nosnap.pref`, so apt can never
   reinstall it, not even as a dependency.
4. Deletes leftover snap directories: `/snap`, `/var/snap`, `/var/lib/snapd`, `/var/cache/snapd`
   and `/root/snap`.

## Tailscale auto-update (`tailscale`)

Turns on Tailscale's auto-update (`tailscale set --auto-update=true`) if it isn't on already.
Maintenance relies on this: it never upgrades Tailscale through apt, because that would cut the
Tailscale SSH session. See [maintenance.md](maintenance.md).

## Ubuntu Pro apt news (`ubuntu_pro`)

Turns off "apt news" (`pro config set apt_news=false`) if it's on: the Ubuntu Pro advertising
shown during `apt upgrade`. The probes aren't attached to Ubuntu Pro and ESM isn't enabled, so
this changes no updates. 26.04 gets normal security updates until 2031.

`ubuntu-pro-client` itself stays installed: Ubuntu's base packages depend on it, and removing it
can take other packages with it.

## Speedtest CLI (`speedtest`)

Installs the official Ookla Speedtest CLI from its release tarball, not from Ookla's apt repo.

1. **Removes** the apt packages `speedtest` (Ookla's) and `speedtest-cli` (an unofficial Python
   version), plus Ookla's apt repo and key files.
2. **Checks the installed version** with `speedtest --version`.
3. **Installs only when needed:** if the binary is missing, isn't `speedtest_version`, or the apt
   package was just removed. The steps:
   - download the tarball for the probe's architecture from `install.speedtest.net`;
   - verify it against the pinned sha256 checksum;
   - extract only the `speedtest` binary;
   - install it to `speedtest_bin_path` (root-owned, mode 0755);
   - delete the temporary files.
4. **Verifies** the binary runs and prints its version.

**Why the tarball:** Ookla rarely releases a new CLI, so the apt repo adds no real updates. It does
add a repo, a key, and a workaround for Ubuntu releases Ookla doesn't publish a repo for. With the
tarball, apt never touches speedtest.

**Why `/usr/bin/speedtest`:** Telegraf on the probes runs that exact path. Removing the apt
package deletes that file, so the role removes the package first and puts the binary back right
after.

**Upgrading speedtest:** set `speedtest_version` and both checksums (the sha256 of each `.tgz`, for
x86_64 and aarch64) in `roles/speedtest/defaults/main.yml`, then run `--tags speedtest`.

In a dry run, the role reports "Would install speedtest …" instead of downloading.

## Settings

In `inventory/group_vars/netprobes/main.yml`. Defaults are in each role's `defaults/main.yml`.

| Setting | Meaning |
|---|---|
| `bootstrap_timezone` | System timezone |
| `bootstrap_install_packages` | Baseline packages to install |
| `bootstrap_remove_packages` | Packages to purge |
| `speedtest_install` | Install the Speedtest CLI |
| `speedtest_version` | Speedtest version to install |
| `speedtest_checksums` | sha256 per architecture for that version |
| `speedtest_bin_path` | Install path; keep `/usr/bin/speedtest` for Telegraf |
