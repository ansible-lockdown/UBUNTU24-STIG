# Ubuntu24STIG

## Based on STIG v1r3

## 1.3.0

2026 May Updates

QA hygiene sweep against V1R3 SCAP canonical benchmark (no rule additions/removals - V1R3 scope preserved).

- meta/main.yml: bump min_ansible_version 2.12.1 -> 2.16.1, set author to Ansible-Lockdown Team, append company suffix to MindPoint Group - A Tyto Athene Company, drop duplicate ubuntuserver galaxy tag
- vars/main.yml: bump min_ansible_version 2.12.1 -> 2.16.1
- LICENSE: fix casing Mindpoint -> MindPoint
- README.md: replace 21 occurrences of UBTU24-STIG with UBUNTU24-STIG so badge and pipeline URLs resolve correctly
- .gitignore: add secret patterns (*.pem *.key *.p12 *.pfx .vault_pass *.vault) and .molecule/ artifact directory
- CHANGELOG.md: renamed from Changelog.md to match Lockdown convention (macOS case-insensitive FS aware rename)
- tasks/main.yml: broaden container detection to also match community.docker.docker connection plugin; convert two absolute-notation file modes to relative ('u=rwx,go=rx' -> 'go-w', 'u-x,go=r' -> 'go-wx')
- tasks/Cat2/UBTU24-70xxxx.yml: UBTU-24-700040 journalctl mode 'u=rwx,g=r,o-rwx' -> 'g-wx,o-rwx' (relative notation)
- tasks/Cat2/UBTU24-10xxxx.yml: UBTU-24-100120 audit step removed shell redirect from ansible.builtin.command (silent failure risk on dash); UBTU-24-100040 title double-space typo collapsed
- tasks/prelim.yml: PRELIM | AUDIT | Discover wireless adapter - reorder register: after changed_when/failed_when
- handlers/main.yml: Audit_immutable_fact notify case fix (change_requires_reboot -> Change_requires_reboot) - case-sensitive handler match was silently dropping the auditd-immutable reboot flag
- .github/workflows/export_badges_public.yml: deleted from private repo (public mirror has its own copy)

CAT 1 - SV-* tag rN-suffix alignment with V1R3 SCAP (rule numbers unchanged, only revision suffix corrected):
- UBTU-24-102000 - SV-270675r1117265 -> SV-270675r1137691
- UBTU-24-600030 - SV-270744r1117272 -> SV-270744r1137699
- UBTU-24-700400 - SV-278917r1135000 -> SV-278917r1155246

CAT 2 - SV-* tag rN-suffix alignment with V1R3 SCAP:
- UBTU-24-100860 - SV-270671r1067118 -> SV-270671r1155244
- UBTU-24-102010 - SV-270676r1068360 -> SV-270676r1155245
- UBTU-24-200270 - SV-274870r1107304 -> SV-274870r1155243
- UBTU-24-600070 - SV-270746r1101769 -> SV-270746r1155242

CAT 3 - SV-* tag rN-suffix alignment with V1R3 SCAP:
- UBTU-24-400340 - SV-270734r1066691 -> SV-270734r1155240
- UBTU-24-600140 - SV-270749r1117267 -> SV-270749r1137695

2026 Feb Updates

- QA fixes: grammar corrections (README.md, defaults/main.yml)
- Removed unused variables: audit_run_heavy_tests, discover_int_uid, ubtu24stig_legacy_boot
- Added missing variable definition: ubtu24stig_time_pool
- Fixed incorrect variable reference: ubtu24stig_sudo_group_required to ubtu24stig_sudo_group_users
- Standardized register variable names in prelim.yml to use prelim_ prefix

2026 Jan Updates

- Added extra options and explanation for audit component
- Linting: removed deprecated parseable option from ansible-lint config
- Improved logic for 300020. Added sudoers_exclude to logic.

CAT 1
- UBTU-24-300038 - ruleid updated
- UBTU-24-700400 - moved from CAT2 rules id updates

CAT 2
- UBTU-24-100110 - ruleid updated
- UBTU-24-100840 - ruleid updated
- UBTU-24-200090 - ruleid updated
- UBTU-24-300039 - ruleid updated
- UBTU-24-700010 - ruleid updated and exclusions
- UBTU-24-901230 - ruleid - audit pkg list updated to remove audisp
- UBTU-24-901240 - ruleid - audit pkg list updated to remove audisp
- UBTU-24-901250 - ruleid - audit pkg list updated to remove audisp
- UBTU-24-909890 - rule moved from 901890 - ruleid updated

CAT 3
- UBTU-24-900950 - fixed conditional logic

## Based on STIG v1r2

## 1.2.0

CAT 1
- UBTU-24-300025 - Updated

CAT 2
- UBTU-24-200020 - ruleid and new steps
- UBTU-24-200040 - requirement change and new tasks
- UBTU-24-200041 - new requirement
- UBTU-24-200042 - new requirement
- UBTU-24-200043 - new requirement
- UBTU-24-200270 - new requirement
- UBTU-24-300006 - updated search path - ruleid
- UBTU-24-300007 - updated search path - ruleid
- UBTU-24-300008 - updated search path - ruleid
- UBTU-24-300009 - updated search path - ruleid
- UBTU-24-300019 - new requirement
- UBTU-24-300020 - new requirement
- UBTU-24-300021 - updated
- UBTU-24-400220 - ruleid

CAT 3
- UBTU-24-200000 - ruleid

### Post-QA continuation (May 2026)

Follow-up work after the initial QA hygiene sweep above, surfaced by the molecule converge cycle and a second-pass review of the audit-paired repo.

Role infrastructure:
- molecule/default/: new scenario added. Uses `geerlingguy/docker-ubuntu2404-ansible:latest` (multi-arch, no Rosetta needed on Apple Silicon), explicit `roles_path` so include_role resolves, `_molecule_role_name` (avoids reserved `role_name`), `setup_audit: true` / `run_audit: true` in host_vars. Prepare playbook installs full STIG package set + `python3-apt`, creates `/run/sshd`, `/etc/sysctl.d`, `/etc/ssh/sshd_config.d`. Converge passes `failed=0`.

tasks/check_prereqs.yml:
- Was unconditionally installing `libselinux-python3` (RHEL-template leakage). Replaced with conditional `python3-apt` installation — the actual apt-module prereq on minimal containers.

README.md Technical Dependencies section:
- `Python3.8` -> `Python 3.10+ (Ubuntu 24.04 ships 3.12)`, `Ansible 2.12+` -> `Ansible 2.16.1+` (matches meta/main.yml), removed `python-def` (non-existent package), `libselinux-python` -> `python3-apt`.

vars/is_container.yml — comprehensive population:
- Initial sweep based on V1R3 SCAP title categorisation (auditd, FIPS, GRUB/kernel/sysctl/modprobe, AppArmor, firewall, time sync, journal_upload).
- Iterative additions surfaced by molecule converge failures: UBTU-24-300041, UBTU-24-600190 (TCP syncookies sysctl), UBTU-24-600200, UBTU-24-600230 (wireless modprobe), UBTU-24-100450, UBTU-24-900920/950/960/980 (manual-only auditd not in V1R3 SCAP), UBTU-24-100660 (apparmor-tagged SSSD).
- Environment-impossible section added: UBTU-24-400340 (SSSD), UBTU-24-600060 (DOD PKI CAs), UBTU-24-600090 (disk encryption), UBTU-24-700300 (CPU NX bit).
- Total: 76 -> 86 -> 90 disables.

Idempotency fixes (molecule converge previously reported changed=7 on first run, changed=6 on second run):
- tasks/prelim.yml — PRELIM apt update: added `changed_when: false`; `cache_valid_time: 7200` is read-only metadata refresh.
- tasks/pre_remediation_audit.yml & tasks/post_remediation_audit.yml — run_audit.sh shell calls had `changed_when: true` forcing 'changed' on every run. Audit is read-only; switched to `changed_when: false`.

Known by-design idempotency violation not fixed:
- templates/etc/ansible/compliance_facts.j2 renders `Benchmark_run_date` and `audit_run_date` timestamps inline, so the rendered file legitimately changes every run. Splitting into stable + volatile fact files is left for a future refactor.

Wrong-toggle gate fix (Phase 5 verification surfaced):
- tasks/Cat2/UBTU24-10xxxx.yml UBTU-24-100650 (SSSD package install, SV-270662) — task was gated by `when: - ubtu24stig_100600` (the sibling rule's toggle, copy-paste artifact) instead of its own `ubtu24stig_100650`. Effect: the `ubtu24stig_100650` toggle was dead and users disabling rule 100650 via its own toggle had no effect (they had to disable the unrelated 100600 toggle, which also disables a different rule). One-line fix: changed `ubtu24stig_100600` -> `ubtu24stig_100650` in the when: clause around line 336.

Dead toggle removed:
- `ubtu24stig_900500` removed from `defaults/main.yml` and `templates/ansible_vars_goss.yml.j2`. Not in V1R3 SCAP, not in V1R5 manual XCCDF — pure placeholder with no corresponding task or audit test. Paired audit-repo removal in vars/STIG.yml.

## Based on STIG v1r1

pre-commit updates
### 1.0.0

### Initial
