# Ansible role-based project (RHEL, AAP)

Five use cases, each one is a **role**. The playbook files in the top folder only
call the role, all the logic and all the settings live inside the role.

```
ansible.cfg
collections/requirements.yml     collections AAP installs on project sync
service_status.yml               Use Case 1
ordered_reboot.yml               Use Case 2
ssh_key_rotation.yml             Use Case 3
system_audit.yml                 Use Case 4
ssh_hardening.yml                Use Case 5
ssh_hardening_rollback.yml       manual rollback for Use Case 5
roles/<name>/defaults/main.yml   every variable (names, limits, paths, mail) is here
roles/<name>/tasks/main.yml      the tasks
roles/<name>/templates/          report templates
```

## How to use in AAP
1. Push this folder to your Git repo and run **Project Sync** (collections are installed from `collections/requirements.yml`).
2. Create one Job Template per playbook. Turn **Privilege Escalation** ON for all of them except `service_status` and `ssh_key_rotation`.
3. Change variables from **Extra Variables** or a **Survey**, no need to edit the role.
4. Always run the first time with **Job Type = Check** and on one staging VM.

| Playbook | Main variables you may pass |
|---|---|
| service_status.yml | `service_status_services` (list), `service_status_fail_on_inactive` |
| ordered_reboot.yml | `ordered_reboot_action` (reboot/shutdown), `ordered_reboot_timeout`, `ordered_reboot_smtp_host`, `ordered_reboot_mail_to` |
| ssh_key_rotation.yml | `ssh_key_rotation_user`, `ssh_key_rotation_new_public_key`, `ssh_key_rotation_new_private_key_path`, `ssh_key_rotation_old_public_key` |
| system_audit.yml | `system_audit_min_os_version`, `system_audit_min_kernel_version`, `system_audit_disk_threshold_percent` |
| ssh_hardening.yml | `ssh_hardening_admin_user`, `ssh_hardening_settings` |

Sequence for Use Case 2: servers are handled one at a time (`serial: 1`) in **inventory order**, so list the hosts in the inventory in the order you want.

## Mentor points and what was done
| Task | Point | Done in |
|---|---|---|
| All | Role-based design | 5 roles, playbooks only call roles |
| 1 | Do not hardcode service names | `service_status_services` chosen by OS family, can be overridden from AAP |
| 2 | Do not reboot directly | `needs-restarting -r` first, reboot only if rc=1 (or `ordered_reboot_force`) |
| 2 | Longer timeout, mail after 10 min | `ordered_reboot_timeout: 600`; on failure `rescue` sends mail, then stops the sequence |
| 3 | Secure passwordless SSH, no private key on nodes | Only the **public** key is installed; private key stays on the control node, its permission is checked (0600/0400); `.ssh` 0700 and `authorized_keys` 0600 are enforced |
| 3 | Safe rotation | backup, install new key, test login with the new key only, revoke old key, test again, automatic rollback on any failure |
| 4 | Explain the `assert` conditions | See the next section |
| 5 | Backup before changing files | `authorized_keys` and `sshd_config` are copied to `.bak_<time>` first; the audit report uses `backup: true` |
| 5 | Rollback | automatic in `rescue` blocks; manual with `ssh_hardening_rollback.yml` |
| 5 | Validate before reload | `validate:` on every edit, then `sshd -t`, then `sshd -T` to confirm the values are really effective, then reload |

## Task 4 - how the assert conditions were identified (for the audit session)
Every condition in `roles/system_audit/tasks/main.yml` reads its limit from
`roles/system_audit/defaults/main.yml`, so nothing is hidden inside the `that:` line.

| Condition | Variable | Why this value | Reference |
|---|---|---|---|
| `distribution == RedHat` | `system_audit_required_distribution` | The project runs RHEL. `ansible_facts['distribution']` returns `RedHat` on RHEL. | Ansible `setup` module facts |
| `distribution_version >= 9` | `system_audit_min_os_version` | RHEL 9 is taken as the minimum supported baseline | Red Hat RHEL 9 documentation |
| `kernel >= 5.14` | `system_audit_min_kernel_version` | RHEL 9.0 ships kernel 5.14, so a RHEL 9 server should never be below it. The text after `-` (for example `-427.el9`) is removed before comparing. | RHEL 9 release (kernel 5.14), e.g. The Register, 10 May 2022 "At last, Red Hat Enterprise Linux 9.0 slips out" |
| `selinux mode == enforcing` | `system_audit_required_selinux_mode` | Standard hardening recommendation | CIS Benchmark for RHEL 9, Red Hat SELinux guide |
| `disk / <= 90%` | `system_audit_disk_threshold_percent` | Same 90% limit that was used in the first version of this playbook | Project requirement |

Ansible documentation used:
- assert: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html
- version test: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/version_test.html
- setup (facts): https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html
- package_facts: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/package_facts_module.html

**Before the audit session:** the thresholds above are a sensible baseline, but if your project or
client has its own baseline document, put its name in the "Reference" column and change the values
in `defaults/main.yml` to match it. Be ready to say which document the numbers came from.

## Important order of work (do not skip)
1. Run `ssh_key_rotation.yml` first (installs the key).
2. In AAP, change the **Machine Credential** to use the SSH private key instead of the password.
3. Only then run `ssh_hardening.yml`. It disables password login, so a job that still uses a password credential cannot connect any more.
   The role also stops by itself if the admin user has no key in `authorized_keys`.

## What was tested and what was not
Tested on a test Linux box with a real sshd, `ansible-lint` (production profile) passed:
service report, system audit (non-compliant case), key rotation (success and automatic rollback),
ssh hardening (success, rollback on a bad value, anti-lockout stop, manual rollback), reboot role
(wrong action check, failure and rescue path).

**Not tested:** a real reboot on RHEL (`dnf-utils` / `needs-restarting` part), a real shutdown, and sending a mail
through your SMTP server. Test those once on a staging VM before using them anywhere important.
