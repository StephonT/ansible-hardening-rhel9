# ansible-hardening-rhel9

**RHCE Ansible Apprenticeship — Project 3**

A custom Ansible role that hardens RHEL 9 systems by managing firewall zones, enforcing SELinux policy, and deploying fail2ban SSH brute-force protection. Built from scratch using `ansible-galaxy init`, the role is fully idempotent, configurable through defaults, and uses the same test/prod promotion workflow from Project 2.

---

## Architecture

```
                        ┌──────────────────────────┐
                        │       Control Node        │
                        │   ansible-hardening-rhel9/ │
                        │   ├── playbooks/           │
                        │   ├── roles/hardening/     │
                        │   ├── group_vars/           │
                        │   └── inventory/            │
                        └───────────────┬────────────┘
                                        │ SSH
               ┌────────────────────────┼────────────────────────┐
               │                        │                         │
    ┌──────────▼──────────┐  ┌──────────▼──────────┐  ┌─────────▼──────────┐
    │     ipaclient1       │  │     ipaclient2       │  │    ipaclient3      │
    │                      │  │                      │  │                    │
    │   [ test group ]     │  │   [ prod group ]     │  │  [ prod group ]   │
    │                      │  │                      │  │                    │
    │  Role applied first  │  │  Promoted only after │  │  Promoted only    │
    │  Idempotency tested  │  │  test passes         │  │  after test passes │
    │  FreeIPA login check │  │                      │  │                    │
    └──────────────────────┘  └──────────────────────┘  └────────────────────┘

  Hardening components applied to every host:
    firewalld  →  Declarative zone + service management
    SELinux    →  Enforcing mode + boolean enforcement
    fail2ban   →  SSH brute-force protection via firewalld rich rules
```

### Inventory Groups

```ini
[test]
ipaclient1

[prod]
ipaclient2
ipaclient3

[hardening_targets:children]
test
prod
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| OS | RHEL 9 with active Red Hat subscription |
| Project 1 | Completed — FreeIPA clients enrolled and accessible |
| Control node | `ansible-core`, `ansible.posix` collection |
| Managed nodes | `firewalld` (pre-installed), `fail2ban` + `fail2ban-firewalld` (installed by role) |
| EPEL | Installed automatically by role via Fedora RPM URL |

---

## Project Structure

```
ansible-hardening-rhel9/
├── ansible.cfg                          # Inventory path, roles path, SSH config
├── inventory/
│   └── hosts.ini                        # test (ipaclient1) and prod (ipaclient2/3)
├── group_vars/
│   └── all.yml                          # Optional overrides for role defaults
├── roles/
│   └── hardening/
│       ├── defaults/
│       │   └── main.yml                 # All configurable variables (overridable)
│       ├── vars/
│       │   └── main.yml                 # Fixed internal variables (not overridable)
│       ├── tasks/
│       │   └── main.yml                 # Firewall, SELinux, and fail2ban tasks
│       ├── handlers/
│       │   └── main.yml                 # Reload firewalld, restart fail2ban
│       ├── templates/
│       │   └── hardening.conf.j2        # fail2ban jail configuration template
│       └── meta/
│           └── main.yml                 # Role metadata and dependencies
├── playbooks/
│   ├── site.yml                         # Master playbook with tag support
│   └── harden.yml                       # Apply hardening role to hardening_targets
└── docs/
    └── idempotency_test.md              # Second-run changed=0 verification
```

---

## Role Variables

### `defaults/main.yml` — Configurable (override in `group_vars` or `host_vars`)

| Variable | Default | Description |
|---|---|---|
| `hardening_firewall_zone` | `public` | firewalld zone to manage |
| `hardening_firewall_services_allowed` | `[ssh, freeipa-ldap, freeipa-ldaps, dns, nfs]` | Services explicitly allowed through the firewall |
| `hardening_firewall_services_blocked` | `[telnet, ftp]` | Services explicitly removed from the zone |
| `hardening_selinux_state` | `enforcing` | SELinux enforcement mode |
| `hardening_selinux_policy` | `targeted` | SELinux policy type |
| `hardening_selinux_booleans` | `[use_nfs_home_dirs: true, ...]` | SELinux booleans to set persistently |
| `hardening_fail2ban_bantime` | `3600` | Ban duration in seconds (1 hour) |
| `hardening_fail2ban_findtime` | `600` | Window for counting failures in seconds |
| `hardening_fail2ban_maxretry` | `5` | Failed attempts before ban |
| `hardening_fail2ban_ignoreip` | `[]` | Extra IPs to never ban (control node IP goes here) |

### `vars/main.yml` — Fixed (internal role constants)

| Variable | Value | Description |
|---|---|---|
| `hardening_packages` | `[fail2ban, fail2ban-firewalld]` | Packages installed by the role |
| `hardening_epel_rpm` | Fedora EPEL 9 RPM URL | EPEL repository package URL |
| `hardening_epel_gpg_key` | Fedora EPEL 9 GPG key URL | GPG key imported before EPEL install |
| `hardening_fail2ban_jail_config` | `/etc/fail2ban/jail.d/hardening.conf` | Rendered jail configuration path |

---

## How It Works

### Firewall

Uses `ansible.posix.firewalld` with `permanent: true` and `immediate: true` to manage services in the target zone. Changes are applied immediately and persist across reboots. A `Reload firewalld` handler fires only when the zone configuration actually changes.

### SELinux

Uses `ansible.posix.selinux` to enforce mode and `ansible.posix.seboolean` to set booleans persistently. If the mode changes (e.g., permissive → enforcing), the role automatically triggers a reboot via the `reboot_required` register variable. `use_nfs_home_dirs: true` is set by default to support the FreeIPA NFS home directory environment from Project 1.

### fail2ban

Installs via EPEL after importing the GPG key with `ansible.builtin.rpm_key` — no `disable_gpg_check` required. The `fail2ban-firewalld` package integrates fail2ban with firewalld by setting `banaction = firewallcmd-rich-rules` globally. Bans are enforced at the firewall layer rather than through iptables directly, which is the correct approach on RHEL 9.

The jail configuration is rendered from `hardening.conf.j2` using `backend = systemd` since SSH logs to journald on RHEL 9. The `ignoreip` line automatically includes localhost and the managed node's own IP:

```jinja2
ignoreip = 127.0.0.1/8 ::1 {{ ansible_facts['default_ipv4']['address'] }}
{% if hardening_fail2ban_ignoreip %} {{ hardening_fail2ban_ignoreip | join(' ') }}{% endif %}
```

---

## Usage

### Install collections
```bash
ansible-galaxy collection install ansible.posix
```

### Run against test group first (always)
```bash
# Check mode — see what would change without touching anything
ansible-playbook playbooks/harden.yml --check --become -l test

# Apply to test
ansible-playbook playbooks/harden.yml --become -l test
```

### Verify test before promoting
```bash
# SELinux enforcing
ansible test -m command -a 'getenforce' --become

# Firewall services
ansible test -m command -a 'firewall-cmd --list-services' --become

# fail2ban sshd jail active
ansible test -m command -a 'fail2ban-client status sshd' --become

# FreeIPA login still works
ssh alice@ipaclient1
```

### Promote to prod
```bash
ansible-playbook playbooks/harden.yml --become -l prod
```

### Run full workflow
```bash
ansible-playbook playbooks/site.yml --become
```

### Run individual components using tags
```bash
ansible-playbook playbooks/site.yml --tags firewall --become
ansible-playbook playbooks/site.yml --tags selinux --become
ansible-playbook playbooks/site.yml --tags fail2ban --become
```

### Test fail2ban + firewalld integration
```bash
# Ban a test IP and verify firewalld received the rich rule
sudo fail2ban-client set sshd banip 192.0.2.1
sudo firewall-cmd --list-rich-rules

# Unban when done
sudo fail2ban-client set sshd unbanip 192.0.2.1
```

---

## Idempotency

Running the role a second time with no system changes produces `changed=0` on every task. This was verified on the test group:

```
PLAY RECAP
ipaclient1 : ok=9  changed=0  unreachable=0  failed=0  skipped=1
```

See `docs/idempotency_test.md` for full test results.

---

## Customizing for Your Environment

To add your control node IP to the fail2ban ignore list, add it to `group_vars/all.yml`:

```yaml
hardening_fail2ban_ignoreip:
  - 192.168.x.x    # your control node IP
```

To add or remove firewall services, override the lists in `group_vars/all.yml`:

```yaml
hardening_firewall_services_allowed:
  - ssh
  - https
  - nfs

hardening_firewall_services_blocked:
  - telnet
  - ftp
  - rsh
```

To disable a specific SELinux boolean, override the list:

```yaml
hardening_selinux_booleans:
  - { name: use_nfs_home_dirs, state: true }
  - { name: httpd_can_network_connect, state: false }
```

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Ansible | Automation engine |
| `ansible.posix.firewalld` | Declarative firewall zone and service management |
| `ansible.posix.selinux` | SELinux mode enforcement |
| `ansible.posix.seboolean` | SELinux boolean management |
| `ansible.builtin.rpm_key` | GPG key import for EPEL |
| firewalld | Host-based firewall |
| SELinux (targeted policy) | Mandatory access control |
| fail2ban + fail2ban-firewalld | SSH brute-force protection via firewalld rich rules |
| Jinja2 | fail2ban jail configuration template |
| EPEL | Repository providing fail2ban on RHEL 9 |
| RHEL 9 | Target OS |

---

## License

MIT