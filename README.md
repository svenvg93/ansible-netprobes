# Ansible: netprobes (Beszel agent, speedtest, maintenance)

| Playbook | What it does |
|----------|--------------|
| `playbooks/bootstrap.yml`   | Timezone, baseline packages, removes unwanted packages and snap (pinned so it can't come back), turns off Ubuntu Pro apt news, Ookla Speedtest CLI on netprobes |
| `playbooks/beszel.yml`      | Installs/updates the Beszel agent via the [`community.beszel.agent`](https://github.com/ansible-collections/community.beszel) role |
| `playbooks/beszel-hub.yml`  | Installs/updates the Beszel hub on the controller (bare metal, `community.beszel.hub` role); needs `-K` |
| `playbooks/maintenance.yml` | Package holds, apt upgrade + autoremove, journal cleanup, reboot if required (25% of hosts at a time), checks the Beszel agent is running |
| `playbooks/site.yml`        | bootstrap → beszel → maintenance |
| `playbooks/release-upgrade.yml` | Upgrade to the next Ubuntu LTS (`do-release-upgrade`), one probe at a time; needs `--limit` |
| `playbooks/reboot.yml`      | Reboots netprobes on demand (25% at a time, or only where required), then checks the Beszel agent is running |

How each part works in detail: [docs/bootstrap.md](docs/bootstrap.md), [docs/beszel.md](docs/beszel.md),
[docs/maintenance.md](docs/maintenance.md), [docs/release-upgrade.md](docs/release-upgrade.md),
[docs/reboot.md](docs/reboot.md).

## Setup

```bash
ansible-galaxy collection install -r requirements.yml
```

1. Netprobes are listed in `inventory/hosts.yml` as `netprobe[01:17]`; adjust the range when probes are added or retired.
2. In the Beszel hub, click **Add System** and copy the public key into `agent_public_key` in `inventory/group_vars/netprobes/main.yml`.
3. Test connectivity: `ansible netprobes -m ping`

Netprobes are managed as `root` over Tailscale SSH, so no `-K` is needed. Only runs that include the controller (`cpe-srv`, as `cpe`) need `-K` for its sudo password.

## Probes

All probes run Ubuntu 26.04 with a single NVMe drive. Numbered by hardware: the i3 boxes first
(01–05), then the N200 boxes (06–17).

| Probe | CPU | RAM | Disk | Network interfaces |
|-------|-----|-----|------|--------------------|
| netprobe01 | Core i3-10100 (4C/8T) | 16 GB DDR4 | 250 GB Samsung 970 EVO Plus | `eno1` 1G (e1000e), `enp1s0f0` + `enp1s0f1` 10G (ixgbe) |
| netprobe02 | Core i3-10100 (4C/8T) | 16 GB DDR4 | 250 GB Samsung 980 | `eno1` 1G (e1000e), `enp1s0f0` + `enp1s0f1` 10G (ixgbe) |
| netprobe03 | Core i3-10100 (4C/8T) | 16 GB DDR4 | 250 GB Samsung 980 | `eno1` 1G (e1000e), `enp1s0f0` + `enp1s0f1` 10G (ixgbe) |
| netprobe04 | Core i3-10100 (4C/8T) | 16 GB DDR4 | 250 GB Samsung 980 | `eno1` 1G (e1000e), `enp1s0f0` + `enp1s0f1` 10G (ixgbe) |
| netprobe05 | Core i3-10100 (4C/8T) | 16 GB DDR4 | 250 GB Samsung 980 | `eno1` 1G (e1000e), `enp1s0f0` + `enp1s0f1` 10G (ixgbe) |
| netprobe06 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe07 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe08 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169), `wlo1` Wi-Fi (iwlwifi, unused) |
| netprobe09 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe10 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe11 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe12 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe13 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169), `wlo1` Wi-Fi (iwlwifi, unused) |
| netprobe14 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe15 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe16 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |
| netprobe17 | N200 (4C/4T) | 8 GB DDR4 | 250 GB Samsung 980 | `enp1s0` 2.5G (r8169) |

Interface speeds are the NIC's maximum, not the negotiated link speed.
To refresh: `ansible netprobes -o -m raw -a 'lscpu | grep "Model name"; ip -br link' </dev/null`.

## Usage

```bash
ansible-playbook playbooks/bootstrap.yml                     # new or rebuilt hosts
ansible-playbook playbooks/bootstrap.yml --tags packages     # just one part: timezone | packages | tailscale | ubuntu_pro | speedtest

ansible-playbook playbooks/beszel-hub.yml -K                              # hub on cpe-srv (install / update)

ansible-playbook playbooks/beszel.yml --ask-vault-pass --check --diff     # dry run
ansible-playbook playbooks/beszel.yml --ask-vault-pass                    # install / update agents
ansible-playbook playbooks/beszel.yml --ask-vault-pass --limit netprobe01 # one host
ansible-playbook playbooks/beszel.yml --ask-vault-pass -e agent_state=absent   # uninstall

ansible-playbook playbooks/maintenance.yml
ansible-playbook playbooks/maintenance.yml -e maintenance_reboot_if_required=false   # never reboot
ansible-playbook playbooks/maintenance.yml -e maintenance_serial=1                   # one host at a time

ansible-playbook playbooks/reboot.yml --limit netprobe01                    # reboot one probe
ansible-playbook playbooks/reboot.yml                                       # all probes, 25% at a time
ansible-playbook playbooks/reboot.yml -e reboot_only_if_required=true       # only where Ubuntu asks for it
```

Agents register themselves with the hub (`agent_hub_url` + universal token in `vault.yml`), so there is no need to add systems by hand.

## Upgrading Beszel

Hub and agents both install the latest release. Upgrade the hub first
(`ansible-playbook playbooks/beszel-hub.yml -K`), then run `ansible-playbook playbooks/beszel.yml`
to bring the agents up to date.

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
- **Hub**: runs bare metal on the controller (`/usr/local/bin/beszel`, data in `/var/lib/beszel`,
  unit `beszel-hub`), listening on `127.0.0.1:8090` only. The Tailscale Service `svc:beszel` on
  cpe-srv publishes it as `https://beszel.tail294b9a.ts.net` (`tailscale serve status`).
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
- **Controller** (`cpe-srv`, the machine Ansible runs on) gets the Beszel hub (`beszel-hub.yml`) and
  maintenance, in a separate play after the netprobes. It is never rebooted by Ansible; when the report says `reboot_required`,
  reboot it by hand. Use `--limit netprobes` to skip it.
