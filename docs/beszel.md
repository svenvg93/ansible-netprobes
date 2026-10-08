# Beszel agent

`playbooks/beszel.yml` installs the [Beszel](https://beszel.dev) monitoring agent on the netprobes
and connects it to the hub. The install itself is done by the
[`community.beszel.agent`](https://github.com/ansible-collections/community.beszel) role from Ansible
Galaxy. This repo supplies the settings and a check before and after.

## When to run it

| Situation | Command |
|---|---|
| New or rebuilt probe | `ansible-playbook playbooks/beszel.yml --ask-vault-pass --limit <probe>` |
| After upgrading the hub | `ansible-playbook playbooks/beszel.yml --ask-vault-pass` |
| After changing a Beszel setting | `ansible-playbook playbooks/beszel.yml --ask-vault-pass` |
| After updating the `community.beszel` collection | dry run first, then the normal run (see "Upgrading") |
| Removing the agent | `ansible-playbook playbooks/beszel.yml --ask-vault-pass -e agent_state=absent` |

Add `--check --diff` to any of these for a dry run.

## How the agent talks to the hub

```
probe (beszel-agent) ──WebSocket over HTTPS──▶ hub (agent_hub_url)
```

- Each agent **connects out** to the hub at `agent_hub_url` and authenticates with the hub's
  **universal token** (`agent_token`). The hub then adds the system itself, so you never add
  probes by hand.
- `agent_public_key` (from the hub's **Add System** dialog) is also required by the role.
- The agent also opens its own SSH listener on `agent_port`, which the hub could use to connect
  in. The collection can't turn that off yet; an upstream PR is planned (see "Upgrading").

**Network requirement:** the probes must be able to reach the hub on port 443. If your tailnet
policy restricts traffic, allow the probes to reach the hub's node.

## What a run does

1. **Pre-check** (this repo): stops with a clear message if the token isn't set in `vault.yml`.
2. **`community.beszel.agent`** role:
   - downloads the latest agent release from GitHub and unpacks it to `/usr/local/bin/beszel-agent`.
     The download runs every time, so the role usually reports "changed" and restarts the agent;
   - creates a `beszel` system user without a login shell;
   - when SMART disks are configured, installs `smartmontools` and gives the agent read access to
     those disks;
   - writes `/etc/systemd/system/beszel-agent.service` with the settings as environment variables
     and systemd sandboxing;
   - enables and starts the service.
3. **Post-check** (this repo): fails the host if `beszel-agent` isn't running afterwards.

## Settings

In `inventory/group_vars/netprobes/main.yml`. The token lives in `vault.yml`.

| Setting | Meaning |
|---|---|
| `agent_public_key` | Public key from the hub's **Add System** dialog |
| `agent_hub_url` | The hub's address; agents connect to it |
| `agent_token` | Universal token. Points to `vault_agent_token` in `vault.yml` |
| `agent_port` | Port of the agent's SSH listener |
| `agent_service_enabled` | Start the agent at boot |
| `agent_smart_disks` | Disks to read SMART data from |

**Choosing `agent_smart_disks`:** use the path `smartctl --scan` reports on a probe. For NVMe
drives that's the controller (`/dev/nvme0`). List the namespace (`/dev/nvme0n1`) as well, so the
service may open both.

Not set on purpose:
- **`agent_version`**: agents always install the latest release.
- **Docker stats**: the `beszel` user isn't in the `docker` group, because that group gives
  effective root.

### Setting up the token

Once, before the first run:

```bash
cp inventory/group_vars/netprobes/vault.yml.example inventory/group_vars/netprobes/vault.yml
# set vault_agent_token to the hub's universal token (hub Settings), then:
ansible-vault encrypt inventory/group_vars/netprobes/vault.yml
```

`vault.yml` is git-ignored. Every `beszel.yml` run then needs `--ask-vault-pass`.

## Upgrading

- **Agents:** upgrade the hub first, then run `beszel.yml`. Agents always get the latest release,
  and an agent newer than the hub may not work with it.
- **The collection:**
  ```bash
  ansible-galaxy collection install -r requirements.yml --upgrade
  ansible-playbook playbooks/beszel.yml --ask-vault-pass --check --diff --limit <probe>
  ```
  Review the diff of the service file, then do a normal run. See "Updating the collections" in
  the README for how the version range works.

## Dry runs

A dry run on a probe that doesn't have the agent yet stops at the role's "start service" task.
A dry run doesn't create the service, so there's nothing to start. That's expected.

## Troubleshooting

```bash
ansible <probe> -m command -a 'systemctl status beszel-agent --no-pager' </dev/null
ansible <probe> -m command -a 'journalctl -u beszel-agent -n 50 --no-pager' </dev/null
```

- **Agent runs but doesn't show up in the hub:** check that the hub is running and that the probe
  can reach it: `curl -sI <agent_hub_url>` from the probe.
- **No SMART data:** compare `smartctl --scan` on the probe with `agent_smart_disks`.
