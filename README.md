# Haven Blueshore — Ansible baseline (Pulse)

Ansible playbooks for Pulse client machines (macOS and Linux). Applied by
**ansible-pull**: each machine checks this repo on a schedule (every 6 hours)
and applies the baseline locally. No control node, no inbound ports — the
machine only needs outbound HTTPS to GitHub and to the Haven Blueshore
installer portal.

## What the baseline does

1. **Zabbix agent** — makes sure the monitoring agent is installed and
   running (fetched from the client's own portal link, pre-configured with
   PSK auto-registration to `mon.havenblueshore.cloud`).
2. **Wazuh agent** — makes sure the security agent is installed and running
   (pre-configured for `siem.havenblueshore.cloud`, self-enrolls on first
   start).
3. **Fleet fleetd** — makes sure the inventory agent is installed and running
   (pre-configured for `inventory.havenblueshore.cloud`, per-client enroll
   secret already baked into the package).
4. **OS patching** — `apt`/`dnf` upgrades on Linux, macOS software updates
   plus Homebrew upgrades on macOS. It never reboots the machine; a pending
   reboot is left for the user.

## No secrets in this repo

This repo is public and contains **no credentials**. The Zabbix PSK, the
Wazuh enrollment password, and the per-client Fleet enroll secret live only
in the installers served from the client's own portal directory — the
playbook downloads those installers at run time and never sees the secrets
as variables. Do not add secrets here.

## How a machine gets enrolled

The client's per-client portal directory (link in their onboarding ticket)
contains a pre-filled setup script:

- Linux: `ansible/setup-ansible-pull-linux.sh`
- macOS: `ansible/setup-ansible-pull-macos.sh`

Running it (as root / with sudo) installs Ansible + git, writes the
scheduled pull (systemd timer on Linux, LaunchDaemon on macOS), and runs the
baseline once immediately. The machine then self-heals on schedule: a
missing agent is re-installed, patches are applied.

The generic scripts on the public portal take the per-client portal URL as
an argument:

```
sudo bash setup-ansible-pull-linux.sh https://install.havenblueshore.cloud/<slug>-<token>/
sudo bash setup-ansible-pull-macos.sh https://install.havenblueshore.cloud/<slug>-<token>/
```

## Variables

| Variable | Required | What it is |
| --- | --- | --- |
| `portal_url` | yes | The client's portal directory URL, no trailing slash. Passed in by the setup script (`ansible-pull -e portal_url=...`). |

## Layout

```
local.yml                  # baseline playbook (ansible-pull runs this)
roles/
  zabbix/tasks/main.yml    # ensure Zabbix agent installed + running
  wazuh/tasks/main.yml     # ensure Wazuh agent installed + running
  fleet/tasks/main.yml     # ensure Fleet fleetd installed + running
  patch/tasks/main.yml     # OS patching (apt / dnf / softwareupdate + brew)
```

## Logs

On the machine: `/var/log/haven-blueshore-ansible-pull.log`.

## The paid alternative

The Tactical RMM agent (Windows today; macOS/Linux if the $55/mo Tier 1
sponsorship is decided) does the same job from one console with real-time
control, patch policies, and scripting. Ansible-pull is the free path: it
automates the same baseline, but there is no central dashboard showing which
machines ran successfully — check the log on the machine, or the agents
themselves (Zabbix / Wazuh / Fleet) to confirm a machine is healthy.
