# Ubuntu24STIG

## 1.5.0 - 2026 August - PAM durability via pam-auth-update

Addresses ansible-lockdown/UBUNTU24-STIG#27. No benchmark version change.

This reverses the decision recorded further down this file, that the PAM rework would stay on the
current release line. Two things changed that judgement:

- **UBTU-24-300028 (CAT I) was not fully remediated on this line.** The task removed `nullok` only
  from `/etc/pam.d/common-password`, but the V1R5 check text greps `common-password` **and**
  `common-auth`. `nullok` originates in the distribution's `unix` profile and reaches `common-auth`,
  which the role never touched, so a host remediated exactly per this line could still fail the
  V1R5 check. The V1R5 and V1R6 check text for this rule is identical; only the fixtext differs, and
  V1R6 corrected it to name both files.
- **The generated files are not a durable place to hold the fix.** Ubuntu builds
  `/etc/pam.d/common-*` from the profiles in `/usr/share/pam-configs` via `pam-auth-update`, which
  any `libpam-runtime` or PAM module upgrade triggers. Edits to the generated files are discarded on
  the next regeneration, silently reinstating a CAT I finding, while a content-only audit that reads
  the generated file reports the host compliant throughout.

Changes:

- UBTU-24-200610, 300016, 300017, 300028 and 400220 now template a profile into
  `/usr/share/pam-configs` and notify a handler running `pam-auth-update --enable`, so the hardening
  is a property of the source rather than of a file rebuilt from under it. Both
  `community.general.pamd` calls are retired.
- New templates `templates/usr/share/pam-configs/{pam_unix,faillock,faildelay,pwquality}.j2`. They
  carry no `file_managed_by_ansible` header: the pam-config format has no comment syntax and
  `pam-auth-update` warns on every such line.
- Relative order in the generated stack is now a property of profile `Priority` - faildelay 384
  above the unix profile 256, faillock 0 below it - rather than of `insertbefore` anchors. The old
  300017 anchored on `auth.*pam_unix.so nullok`, an anchor that 300028 removes, so the two controls
  were order-coupled through the generated file.
- `pam-auth-update --enable` is used rather than `--force`. `--force` overrides the safety check that
  refuses to discard manual edits to the generated files, and discarding those silently is the
  behaviour these controls are moving away from.
- UBTU-24-400220 gains `rounds` on the `pam_unix` password line via `ubtu24stig_passwd_rounds`
  (default 100000). The V1R5 check-content is a finding when `rounds` is below 100000, so this is the
  STIG floor rather than a tuning knob.
- UBTU-24-400020 deliberately keeps its `lineinfile`: it is gated behind
  `ubtu24stig_uses_smartcard` and `ubtu24stig_disruption_high`, and there is no
  distribution-provided pam-config for `pam_pkcs11`.
- The superseded faillock block written into `common-auth` by earlier releases is now removed by
  marker. That marker is reproduced byte for byte, including its misspelled company name, because
  `blockinfile` matches markers exactly - correcting the spelling would leave a duplicate faillock
  stanza on every host that still carries the old block.
- New: `ubtu24stig_pam_confd_dir`, `ubtu24stig_pam_pwunix_file`, `ubtu24stig_pam_faillock_file`,
  `ubtu24stig_pam_faildelay_file`, `ubtu24stig_pam_pwquality_file`, `ubtu24stig_pwquality_retry`,
  `ubtu24stig_passwd_rounds`.
- `molecule/default/verify.yml` now asserts PAM stack ordering and re-asserts the hardening after a
  forced regeneration, so a future change that reverts to editing the generated files fails the
  suite. Its previous header forbade exactly these assertions on this line; that note described the
  old design and has been rewritten.

## 1.5.0 - 2026 August - Contributing guide and README refresh

- replaced `CONTRIBUTING.rst` with `CONTRIBUTING.md`, carrying the current Ansible-Lockdown
  contributing guide. It adds the approved-contributor model for pull requests, keeps issues open to
  everyone, and retains the DCO 1.1 text and the GPG plus Signed-off-by requirements
- `README.md`: added a Contributing section pointing at `CONTRIBUTING.md`, and refreshed the
  Community Contribution section, which still described the previous open pull request workflow and
  so contradicted the new guide
- `README.md`: removed the decorative emoji from headings, and switched the social badge from
  `twitter.com` to `x.com`
- `molecule/README_Molecule_QuickStart.md` and `molecule/default/molecule.yml` no longer name this
  repository. The role resolves itself from the basename of `MOLECULE_PROJECT_DIRECTORY`, so these
  were documentation only, and the public mirror previously inherited a guide telling readers to
  change into a repository they cannot clone
- `molecule/README_Molecule_QuickStart.md`: the audit-branch section placed `audit_git_version` in
  `vars/audit.yml`, which no longer exists, and claimed `include_vars` precedence meant only
  extra-vars could override it. It is an ordinary role default in `defaults/main/audit.yml`, so
  inventory, group and play vars work - as this changelog already records
- `molecule/default/converge.yml`: dropped the stale V1R3 release marker from the comment header rather than bumping it, since a hardcoded release in a comment goes stale every cycle

## 1.5.0 - 2026 August Updates - kdump unit, CCI alignment

- `UBTU-24-600070` masked `kdump` in `tasks/Cat2/UBTU24-60xxxx.yml`. The unit shipped on Ubuntu is `kdump-tools.service`; `kdump.service` does not exist, and `masked: true` against a nonexistent unit creates a `/dev/null` symlink and reports success, so the control was a silent no-op. Verified in a container: masking `kdump` leaves `systemctl status kdump-tools.service` reporting "could not be found", which the check text treats as a finding, while masking `kdump-tools` reports `Loaded: masked` as required. The `stop` task keeps its `linux-crashdump` package gate; the `mask` task stays ungated so a later install cannot start the service
- CCI tags aligned with the benchmark across 31 controls: 15 carried the `CCI-000000` placeholder and 16 were missing a CCI the XCCDF lists. Tags were regenerated from this branch's own XCCDF revision rather than copied from the other benchmark line
- two task names carried `PATCH |Ubuntu` with no space after the separator (`tasks/Cat2/UBTU24-60xxxx.yml`, `tasks/Cat2/UBTU24-901xxx.yml`), which breaks the `SEV | ID | PHASE | title` convention any parser relies on
- `.gitignore` no longer ignores `.github/`. The directory was ignored while the workflows and `dependabot.yml` were all tracked, so every new file there had to be force-added
- the role defaults moved from a single `defaults/main.yml` to a `defaults/main/` directory holding `main.yml` and `audit.yml`, and every audit variable is now consolidated in `defaults/main/audit.yml`. `vars/audit.yml` is removed along with the `include_vars` that loaded it. This is a variable-precedence change and that is the point: those settings were previously loaded by `include_vars`, which outranks play and host vars, so only extra-vars could override them. They are now ordinary role defaults, so inventory and play vars work. Verified by setting `audit_git_version` from a play var and confirming it takes effect, which it could not before
- the Repo QA Checker pin moved to `2.8.4`, which is the first release that accepts a `defaults/main/` directory. Earlier versions look only for `defaults/main.yml` and abort before running a single check


## 1.5.0 - 2026 August - Company name updated to Quantum Sky

- the parent company name changed from Tyto Athene to Quantum Sky. Renamed in `meta/main.yml`, `vars/main.yml` and `LICENSE`. `vars/main.yml` feeds `file_managed_by_ansible`, so the header written into every templated file carries the new name; expect a one-off change on those files at the next run
- the Repo QA Checker pin moved from `2.8.1` to `2.8.3`. `2.8.1` expected the former parent company in `meta/main.yml`, so it failed this branch on its own rename; `2.8.3` expects the new name. Its Company Naming check now also reads the `UBTU-24-200610` marker noted below, which cannot be renamed, so that single finding is recorded in `.qa_baseline.json`
- deliberately not renamed: the `UBTU-24-200610` `blockinfile` marker. It is matched byte for byte to remove the block written by an earlier release, so changing any character in it would leave a duplicate faillock stanza on every host that still has the old block
- deliberately not renamed: existing entries in this file, which record what was true when written
## 1.5.0 - 2026 August Alignment - CI action versions and documentation

Alignment pass at the existing V1R5 release (no benchmark version change). This line receives the documentation and CI work; the PAM rework was at this point deliberately confined to the current release line, since it changes authentication behaviour and this branch is in maintenance. That decision was later reversed - see the PAM durability entry at the top of this file for why.

- `actions/checkout` moved from `@v6.0.2` to `@v7` across the devel and main pipeline workflows, all four references. `@v7` is a moving pointer rather than a frozen pin: upstream advances the major tag onto each release, so future v7.x patches are picked up with no maintenance and only a major would need review.
- Added the Molecule and Repo QA workflows, matching the current release line so both maintained lines get the same gates. Branch filters list both repository models (`latest`/`benchmark*` and `devel`/`main`) so the files need no edit when overlaid to the public mirror, the same convention the pipeline workflows already use.
- Added `molecule/default/verify.yml`, which this line previously lacked. Without it `molecule verify` exits 0 on a missing playbook, so the workflow would have reported success without asserting anything. It carried, at this point, only the checks this line then implemented - the SSH hardening and login-banner spot-checks. The current release line's PAM assertions were deliberately **not** carried back, because this line still edited `/etc/pam.d` directly and shipped no pam-configs profiles, so those assertions would have failed against a correctly remediated host. That premise no longer holds - see the PAM durability entry at the top of this file.
- Added `.qa_baseline.json` for the Repo QA workflow, absorbing the single known audit-variable warning.
- No Dependabot configuration is added here. Dependabot reads its configuration from the repository default branch only, so the file would have no effect on this branch and would simply become a merge conflict later.
- README: the DISA download link returned 404. Corrected to the published path and verified to return 200.
- README: removed the emoji from the section headings and the check-mode caution bullet, leaving the wording unchanged.
- `templates/lockdown_audit.yml.j2` gained the `file_managed_by_ansible` header, leaving `templates/etc/issue.net.j2` as the only deliberate exclusion since it renders the mandated Standard Mandatory DOD Notice and Consent Banner wording, which must not be prefixed.
- Inlined the single-item `when:` list in the prelim wireless-adapter task. The remaining block-form condition holds a multi-line `or` chain where a list is clearer and is left alone.
- Corrected a `BENCMHARK_TYPE` typo in a `vars/audit.yml` comment.
- The goss upstream references in the README and the molecule QuickStart are left pointing at the original project on this line, because `vars/audit.yml` here still resolves the binary from that upstream. Changing the documentation without the source would misdescribe the branch.

## 1.5.0 - 2026 July QA

QA pass at the existing V1R5 release (no benchmark version change):

- Guarded the three auditd handlers (`Auditd_immutable_check`, `Audit_immutable_fact`, `Restart_auditd`) with `when: not system_is_container` so they no-op on containerized targets instead of failing. On `Audit_immutable_fact` the container guard is ordered before the `discovered_auditd_immutable_check.stdout` test so that fact reference is never evaluated when the check handler was skipped.
- Switched `Restart_auditd` from `ansible.builtin.command: service auditd restart` to the `ansible.builtin.systemd_service` module (`state: restarted`), and migrated the remaining `ansible.builtin.systemd` handler calls to `ansible.builtin.systemd_service` (`Systemd_daemon_reload`, `Restart_ssh`, `Restart_chrony`, `Restart_sssd`, `Restart_rsyslog`) for consistency with `Restart_gdm3`.
- Added `prompt.md` and `test_inv` to `.gitignore`.
- Bumped pre-commit hook pins: gitleaks `v8.29.1` -> `v8.30.1`, ansible-lint `v25.11.0` -> `v26.3.0`.
- Normalized the `audit_results` block-scalar indentation in `vars/audit.yml` (style only; renders identically).
- UBTU-24-300016 (SV-270705): enforced `pam_pwquality.so retry=3` in `/etc/pam.d/common-password` via `community.general.pamd`. The rule previously set only `enforcing = 1` in `/etc/security/pwquality.conf`; the STIG also requires the `retry` value in `common-password` to be 1-3, so a system with a non-compliant `retry` was not corrected.
- UBTU-24-600130: fixed the non-disruptive WARN branch, which guarded on an undefined `sudo_group_members_not_required` and so never fired. It now uses the registered `discovered_sudo_group_users` filtered by `difference(ubtu24stig_sudo_group_users)`, and only warns when extra sudo-group members actually exist.
- UBTU-24-200090: corrected the task name identifier from `UBTU-24_200090` (underscore) to `UBTU-24-200090`; tag, toggle, and rule IDs were already correct, so this was a display-string fix only.
- Moved `audit_output_collection_method` and `audit_output_destination` from `vars/audit.yml` to `defaults/main.yml`. Loaded via `include_vars`, `vars/audit.yml` sits above inventory in variable precedence, so consumers could only override these two via `--extra-vars`/`set_fact`; in `defaults/main.yml` they are overridable through standard `group_vars`/`host_vars`. Default values are unchanged.

## 1.5.0 - 2026 June QA

QA pass at the existing V1R5 release (no benchmark version change):

- Fixed UBTU-24-400360 and UBTU-24-400340 to notify the `Restart_sssd` handler so `/etc/sssd/sssd.conf` changes take effect without a manual restart.
- Removed 11 unused handlers - 9 dead (Build_aide_db, Reload_sysctl, six Remount_*, Update_aliases) and 2 OS leakage not applicable to Ubuntu (Firewalld_reload, Restart_NetworkManager).
- Added `molecule/README_Molecule_QuickStart.md` and linked it from the README.
- Molecule: fetch audit output to the host (`fetch_audit_output: true` + `audit_output_destination`) so pre/post goss scans are reviewable after a converge.
- Converted bare `ansible_*` magic-fact references to the `ansible_facts['...']` form for the Ansible 2.19 deprecation - `ansible_env.SUDO_USER` and `ansible_virtualization_type` in `tasks/main.yml`.
- Added a `| default('connecting user')` fallback to the SUDO_USER reference in the connecting-user password-check task name so the task displays cleanly (and no longer emits a Jinja template error) when run directly as root with no `SUDO_USER` in the environment.
- Added `set -o pipefail` and `args.executable` to every `ansible.builtin.shell` task for pipefail-safe shell behavior, with a new `ubtu24stig_shell_executable` variable (default `/bin/bash`) in `vars/main.yml`.
- New Alignment Strategy: a standardized, OS-agnostic layout for the audit framework - the audit-vars bridge template, a dedicated `audit_results` fact template, audit configuration variables consolidated in `vars/audit.yml`, and bracket-notation `ansible_facts` access - adopted to keep the audit plumbing consistent and maintainable going forward. The three entries below implement it:
- Audit bridge rework (New Alignment Strategy): renamed `templates/ansible_vars_goss.yml.j2` -> `templates/lockdown_audit.yml.j2`; split the `[lockdown_audit_details]` section out of `compliance_facts.j2` into a new `templates/etc/ansible/audit_results.j2` rendered to `audit_results.fact`, with a more robust `audit_summary` (`post_audit_results | default(pre_audit_results)`, guarded on both being defined); added `pre_audit_results is defined or post_audit_results is defined` guards to the audit-fetch and summary-display tasks.
- Converted all `ansible_facts.<key>` dot-notation references to bracket notation `ansible_facts['<key>']` across tasks, handlers and vars (e.g. `ansible_facts.packages` -> `ansible_facts['packages']`, `ansible_facts.date_time.epoch` -> `ansible_facts['date_time']['epoch']`), as part of the New Alignment Strategy. Notation-only change (the dot form was never deprecated); bracket is the recommended, collision-safe form for dictionary access.
- Relocated audit configuration variables from `defaults/main.yml` to `vars/audit.yml` as part of the New Alignment Strategy: `audit_max_concurrent`, `get_audit_binary_method`, `audit_bin_validate_certs`, `audit_bin_copy_location`, `audit_content`, `audit_conf_source`, `audit_conf_dest`, `audit_output_collection_method`, `audit_output_destination`, `audit_log_dir`. Only `fetch_audit_output` remains in `defaults/main.yml` (it gates the always-evaluated "Fetch audit files" task, so it must be defined even when `vars/audit.yml` is not loaded). Relaxed the `vars/audit.yml` include gate to `when: run_audit or audit_only` (dropping the `setup_audit` requirement) so the relocated vars are loaded whenever the audit runs. Note: with the vars now in `vars/audit.yml` (`include_vars`, precedence 18), `--extra-vars` is the only inventory-level way to override them; molecule sets `audit_output_destination` via `set_fact` (precedence 19) to drive the audit-output fetch.
- Removed the orphaned `ubtu24stig_time_pool` variable (its only consumer, `templates/etc/chrony/sources.d/pool.sources.j2`, was removed earlier this cycle; `server.sources.j2` uses `ubtu24stig_time_synchronization_servers`).
- QA hygiene fixes: wired the orphaned `check_prereqs.yml` (python3-apt prerequisite) into `tasks/main.yml`; corrected tag drift on 7 controls (UBTU-24-300023/600000/600010/600060 `SV-` -> `V-` rule-id tag, added missing `CAT2` tag to UBTU-24-400320/900040, fixed `RG-OS` -> `SRG-OS` on UBTU-24-901220); removed orphaned `templates/etc/aide.conf.j2` (unused, carried undefined UB22 variables) and `templates/etc/chrony/sources.d/pool.sources.j2`; added `.gitignore` patterns.
- Corrected rule-id tag values on 5 controls (verified against the V1R5 XCCDF): UBTU-24-300027 `V-260573` -> `V-270713`, UBTU-24-100600 `V-260478` -> `V-270661` (both were leaked RHEL-style V-IDs), UBTU-24-400020 `V-2670721` -> `V-270721` (extra digit), UBTU-24-500050 `V-27074` -> `V-270741` (truncated), and UBTU-24-900780 long-form tag `SV-270780r1066829_rulee` -> `SV-270780r1066829_rule`. These tag-value typos broke selective `--tags V-...` runs and rule-to-audit ID mapping.
- Fixed the UBTU-24-200650 / UBTU-24-200660 GDM banner rule mapping: each rule ID now writes the dconf option its V1R5 XCCDF fixtext specifies - UBTU-24-200650 (SV-270692, "must enable") sets `banner-message-enable: true`; UBTU-24-200660 (SV-270693, "must display") sets `banner-message-text`. The title and `option:` were previously crossed between the two rules. System end-state was already correct (both run), so this corrects the per-rule `--tags`/audit ID-to-action mapping, not the hardened result.

## Based on STIG v1r5

## 1.5.0

2026 May Updates

DISA released V1R5 of the Ubuntu 24.04 LTS STIG on 01 April 2026. ALD did NOT cut a V1R4 release branch - this upgrade covers the cumulative V1R3 -> V1R5 changeset in one cycle. Source-of-truth diff is the V1R5 Manual XCCDF (194 rules) against the V1R3 defaults toggle list.

Benchmark version strings:
- defaults/main.yml: `benchmark_version` `v1.3.0` -> `v1.5.0`
- templates/ansible_vars_goss.yml.j2: `ubtu24stig_100050` toggle added; `ubtu24stig_300024` toggle removed.

Rule additions (V1R3 -> V1R5 union):
- UBTU-24-100050 (CAT I, high) - NEW in V1R5. Ubuntu 24.04 LTS must not have the `nfs-kernel-server` package installed. Patch removes both `nfs-common` and `nfs-kernel-server` packages.

Rule removals (V1R3 -> V1R5 union):
- UBTU-24-300024 (CAT III, low) - REMOVED in V1R5. Previously required showing date and time of last successful logon via `pam_lastlog`. Task and toggle deleted.

SV-* tag drift (8 rules, V1R3 -> V1R5 cumulative):
- UBTU-24-100110 - SV-270650r1134802 -> SV-270650r1155241
- UBTU-24-300025 - SV-270711r1101772 -> SV-270711r1184069
- UBTU-24-600150 - SV-270750r1117267 -> SV-270750r1137695
- UBTU-24-700020 - SV-270757r1066760 -> SV-270757r1184072
- UBTU-24-700060 - SV-270761r1067180 -> SV-270761r1184074
- UBTU-24-700070 - SV-270762r1066775 -> SV-270762r1184076
- UBTU-24-700080 - SV-270763r1066778 -> SV-270763r1184078
- UBTU-24-700090 - SV-270764r1066781 -> SV-270764r1184080

V1R5 Fix-text content updates:
- UBTU-24-700020 - journal permissions aligned to Canonical recommendation: dropped the setgid bit from `2640` to `0640` for `/run/log/journal`, `/run/log/journal/%m`, `/var/log/journal`, and `/var/log/journal/%m`. The `system.journal` entry was already at `0640`.

Pre-existing findings fixed during V1R5 cycle (carried forward from V1R3 state):
- F-003: UBTU-24-200320 - SV-* tag had a typo `S-270688r1066553_rule` (missing the leading V). Corrected to `SV-270688r1066553_rule`.
- F-004: UBTU-24-400360 - toggle existed in defaults but the task body was missing entirely. Added a Lockdown-style PATCH task that configures the SSSD PKI entries via `community.general.ini_file` when `/etc/sssd/sssd.conf` exists, and emits a WARN when SSSD is not configured (PKI-via-SSSD configuration is environment-specific).

Style:
- tasks/Cat3/UBTU24-300xxx.yml - dropped trailing blank line left behind by the UBTU-24-300024 removal (yamllint clean).

QA hygiene - register variable naming (resolves 14 duplicate-register WARN findings in `Ansible_Lockdown_QA_Repo_Check.py` v2.7.0 Variable Naming check):
- tasks/Cat2/UBTU24-30xxxx.yml - UBTU-24-300013 block register renamed `discovered_system_bin_files_owner` -> `discovered_system_bin_files_group_owner` (the UBTU-24-300012 block keeps the original name).
- tasks/Cat2/UBTU24-70xxxx.yml - UBTU-24-700140 register renamed -> `discovered_var_log_syslog_owner`; UBTU-24-700150 register renamed -> `discovered_var_log_syslog_perms`; UBTU-24-700310 register renamed `discovered_sysctl_conf_files` -> `discovered_kernel_sysctl_conf_files` (resolves cross-file collision with Cat2/UBTU24-60xxxx.yml:205).
- tasks/Cat2/UBTU24-900xxx.yml - UBTU-24-900050 register renamed -> `discovered_audit_files_owner`; UBTU-24-900060 register renamed -> `discovered_audit_files_group`.
- tasks/Cat2/UBTU24-901xxx.yml - UBTU-24-901240 register renamed -> `discovered_audit_tools_owner`; UBTU-24-901250 register renamed -> `discovered_audit_tools_group`; UBTU-24-901310 register renamed -> `discovered_auditd_logfile_owner`; UBTU-24-901380 register renamed -> `discovered_auditd_logfile_dir`.
- tasks/Cat3/UBTU24-600xxx.yml - UBTU-24-600140 register renamed `discovered_sysctl_conf_files` -> `discovered_kernel_msg_sysctl_conf_files` (resolves cross-file collision with Cat2/UBTU24-60xxxx.yml:205).
- tasks/Cat3/UBTU24-90xxxx.yml - UBTU-24-900920 register renamed `discovered_auditd_logfile` -> `discovered_auditd_logfile_space` (resolves cross-file collision with Cat2/UBTU24-901xxx.yml:226).
- tasks/pre_remediation_audit.yml - JSON branch register `pre_audit_summary` -> `pre_audit_summary_json`; documentation branch register `pre_audit_summary` -> `pre_audit_summary_documentation`. Branches are mutually exclusive via `when: audit_format == "json"` vs `"documentation"`, so this rename is cosmetic but clears the QA WARN.
- tasks/post_remediation_audit.yml - same pattern: JSON branch -> `post_audit_summary_json`; documentation branch -> `post_audit_summary_documentation`.
- tasks/main.yml - gate at lines 143-144 ("Add ansible file showing Benchmark and levels applied...") retargeted from `post_audit_summary is defined` to `post_audit_results is defined`. `post_audit_results` is set as a fact in both branches of post_remediation_audit.yml, so gate behavior is preserved across the JSON/doc rename split.

QA hygiene - manual warn-count vars placement (resolves 27 Manual Warn Count WARN findings in `Ansible_Lockdown_QA_Repo_Check.py` v2.7.0):
- Moved `vars: { warn_control_id: ... }` off the parent block and onto the inner task that imports `warning_facts.yml`, so the var sits at the same indent as `ansible.builtin.import_tasks:` (matches the canonical pattern already used in `tasks/fetch_audit_output.yml`).
- Files touched: tasks/Cat1/UBTU24-30xxxx.yml, tasks/Cat1/UBTU24-60xxxx.yml, tasks/Cat2/UBTU24-30xxxx.yml, tasks/Cat2/UBTU24-40xxxx.yml, tasks/Cat2/UBTU24-60xxxx.yml, tasks/Cat2/UBTU24-70xxxx.yml, tasks/Cat2/UBTU24-901xxx.yml, tasks/Cat3/UBTU24-90xxxx.yml.
- tasks/Cat2/UBTU24-60xxxx.yml - UBTU-24-600090 has two `warning_facts.yml` imports (encrypted vs unencrypted partition paths); task-level `warn_control_id: 'UBTU-24-600090'` added to both.
- tasks/Cat2/UBTU24-60xxxx.yml - UBTU-24-600230 had a stray block-level `warn_control_id: 'MEDIUM | UBTU-24-600230 '` (trailing space) with no `warning_facts.yml` import in its block; this dead setting was removed entirely.

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
- Was unconditionally installing `libselinux-python3` (RHEL-template leakage). Replaced with conditional `python3-apt` installation - the actual apt-module prereq on minimal containers.

README.md Technical Dependencies section:
- `Python3.8` -> `Python 3.10+ (Ubuntu 24.04 ships 3.12)`, `Ansible 2.12+` -> `Ansible 2.16.1+` (matches meta/main.yml), removed `python-def` (non-existent package), `libselinux-python` -> `python3-apt`.

vars/is_container.yml - comprehensive population:
- Initial sweep based on V1R3 SCAP title categorisation (auditd, FIPS, GRUB/kernel/sysctl/modprobe, AppArmor, firewall, time sync, journal_upload).
- Iterative additions surfaced by molecule converge failures: UBTU-24-300041, UBTU-24-600190 (TCP syncookies sysctl), UBTU-24-600200, UBTU-24-600230 (wireless modprobe), UBTU-24-100450, UBTU-24-900920/950/960/980 (manual-only auditd not in V1R3 SCAP), UBTU-24-100660 (apparmor-tagged SSSD).
- Environment-impossible section added: UBTU-24-400340 (SSSD), UBTU-24-600060 (DOD PKI CAs), UBTU-24-600090 (disk encryption), UBTU-24-700300 (CPU NX bit).
- Total: 76 -> 86 -> 90 disables.

Idempotency fixes (molecule converge previously reported changed=7 on first run, changed=6 on second run):
- tasks/prelim.yml - PRELIM apt update: added `changed_when: false`; `cache_valid_time: 7200` is read-only metadata refresh.
- tasks/pre_remediation_audit.yml & tasks/post_remediation_audit.yml - run_audit.sh shell calls had `changed_when: true` forcing 'changed' on every run. Audit is read-only; switched to `changed_when: false`.

Known by-design idempotency violation not fixed:
- templates/etc/ansible/compliance_facts.j2 renders `Benchmark_run_date` and `audit_run_date` timestamps inline, so the rendered file legitimately changes every run. Splitting into stable + volatile fact files is left for a future refactor.

Wrong-toggle gate fix (Phase 5 verification surfaced):
- tasks/Cat2/UBTU24-10xxxx.yml UBTU-24-100650 (SSSD package install, SV-270662) - task was gated by `when: - ubtu24stig_100600` (the sibling rule's toggle, copy-paste artifact) instead of its own `ubtu24stig_100650`. Effect: the `ubtu24stig_100650` toggle was dead and users disabling rule 100650 via its own toggle had no effect (they had to disable the unrelated 100600 toggle, which also disables a different rule). One-line fix: changed `ubtu24stig_100600` -> `ubtu24stig_100650` in the when: clause around line 336.

Dead toggle removed:
- `ubtu24stig_900500` removed from `defaults/main.yml` and `templates/ansible_vars_goss.yml.j2`. Not in V1R3 SCAP, not in V1R5 manual XCCDF - pure placeholder with no corresponding task or audit test. Paired audit-repo removal in vars/STIG.yml.

## Based on STIG v1r1

pre-commit updates
### 1.0.0

### Initial
