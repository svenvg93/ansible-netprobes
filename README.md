# Ansible: netprobes (Beszel agent, speedtest, maintenance)

| Playbook | What it does |
|----------|--------------|
| `playbooks/bootstrap.yml`   | Timezone, baseline packages, removes unwanted packages and snap (pinned so it can't come back), turns off Ubuntu Pro apt news, Ookla Speedtest CLI on netprobes |
| `playbooks/beszel.yml`      | Installs/updates the Beszel agent via the [`community.beszel.agent`](https://github.com/ansible-collections/community.beszel) role |
| `playbooks/maintenance.yml` | Package holds, apt upgrade + autoremove, journal cleanup, reboot if required (25% of hosts at a time), reports failed units, checks the Beszel agent is running |
| `playbooks/site.yml`        | bootstrap → beszel → maintenance |
| `playbooks/release-upgrade.yml` | Upgrade to the next Ubuntu LTS (`do-release-upgrade`), one probe at a time; needs `--limit` |

How each part works in detail: [docs/bootstrap.md](docs/bootstrap.md), [docs/beszel.md](docs/beszel.md),
[docs/maintenance.md](docs/maintenance.md), [docs/release-upgrade.md](docs/release-upgrade.md).

## Setup

```bash
ansible-galaxy collection install -r requirements.yml
```

1. Netprobes are listed in `inventory/hosts.yml` as `netprobe[01:16]`; adjust the range when probes are added or retired.
2. In the Beszel hub, click **Add System** and copy the public key into `agent_public_key` in `inventory/group_vars/netprobes/main.yml`.
3. Test connectivity: `ansible netprobes -m ping`

Netprobes are managed as `root` over Tailscale SSH, so no `-K` is needed. Only runs that include the controller (`cpe-srv`, as `cpe`) need `-K` for its sudo password.

## Usage

```bash
ansible-playbook playbooks/bootstrap.yml                     # new or rebuilt hosts
ansible-playbook playbooks/bootstrap.yml --tags packages     # just one part: timezone | packages | tailscale | ubuntu_pro | speedtest

ansible-playbook playbooks/beszel.yml --ask-vault-pass --check --diff     # dry run
ansible-playbook playbooks/beszel.yml --ask-vault-pass                    # install / update agents
ansible-playbook playbooks/beszel.yml --ask-vault-pass --limit netprobe01 # one host
ansible-playbook playbooks/beszel.yml --ask-vault-pass -e agent_state=absent   # uninstall

ansible-playbook playbooks/maintenance.yml
ansible-playbook playbooks/maintenance.yml -e maintenance_reboot_if_required=false   # never reboot
ansible-playbook playbooks/maintenance.yml -e maintenance_serial=1                   # one host at a time
```

Agents register themselves with the hub (`agent_hub_url` + universal token in `vault.yml`), so there is no need to add systems by hand.

## Upgrading Beszel

Agents install the latest release. Bump the hub image tag in `~/docker/beszel/docker-compose.yml`
first, then run `ansible-playbook playbooks/beszel.yml` to bring the agents up to date.

## Updating the collections

The collections in `requirements.yml` are not updated automatically; `install` keeps whatever
version is already there. To pull in a new release (e.g. once a PR to `community.beszel` ships):

```bash
ansible-galaxy collection list community.beszel                 # installed version
ansible-galaxy collection install -r requirements.yml --upgrade # newest allowed by requirements.yml
ansible-playbook playbooks/beszel.yml --ask-vault-pass --check --diff --limit netprobe01   # review changes
```

`community.beszel` is pinned to `>=2.0.0,<3.0.0`, so `--upgrade` never jumps to a new major version.
To require a release with a feature you need, raise the lower bound (e.g. `">=2.1.0,<3.0.0"`).
For a major version, read the collection's changelog first, then change the range.

## Notes

- Docker container stats are intentionally off (the `beszel` user is not in the `docker` group).
- The agent is installed to `/usr/local/bin/beszel-agent`. Hosts previously set up with the
  official install script also have `/opt/beszel-agent`; the service file gets replaced, so the
  old directory is just leftover and can be removed.
- Agents connect out to the hub at `https://beszel.tail294b9a.ts.net` (port 443), so your tailnet
  ACL must allow the netprobes to reach the hub's node.
- **Speedtest**: installed from Ookla's release tarball (pinned version + sha256 in
  `roles/speedtest/defaults/main.yml`) to `/usr/bin/speedtest`, the path Telegraf calls. The old apt
  package and Ookla apt repo are removed, so apt never touches it. To upgrade, bump
  `speedtest_version` and the checksums. Run it as `speedtest --accept-license --accept-gdpr` in scripts.
- **Package holds** (`maintenance_hold_packages`) are only ever added; release one with
  `sudo apt-mark unhold <pkg>`.
- **Tailscale is never upgraded by maintenance.** Upgrading it restarts `tailscaled`, which drops the
  Tailscale SSH session mid-apt. It is held only while apt upgrades (`maintenance_skip_upgrade_packages`)
  and released afterwards, so Tailscale's own auto-update (enabled by bootstrap, `--tags tailscale`)
  keeps it current. If you ever need to force it: `tailscale update` on the host, or
  `ansible netprobes -m command -a 'systemd-run --collect tailscale update --yes'`.
- **Maintenance fails a host** if `beszel-agent` is installed but not running afterwards
  (`maintenance_verify_beszel: false` to turn that off).
- Bootstrap settings (timezone, packages to remove) live in `inventory/group_vars/netprobes/main.yml`.
- **Controller** (`cpe-srv`, the machine Ansible runs on) only gets maintenance, in a separate play
  after the netprobes. It is never rebooted by Ansible; when the report says `reboot_required`,
  reboot it by hand. Use `--limit netprobes` to skip it. A Docker upgrade there restarts the Beszel hub.
