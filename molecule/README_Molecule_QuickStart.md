# Molecule Quick Start - Ubuntu 24.04 LTS STIG

> Audience: anyone new to molecule testing with this Ansible Lockdown role. Read top-to-bottom the first time; come back to the reference tables later.

## What this scenario does

This `molecule/` directory ships one scenario: `default/`. It spins up a throwaway Ubuntu 24.04 LTS Docker container, applies the **whole** Ansible Lockdown Ubuntu 24.04 STIG role against it, then runs the paired goss audit (from the `UBUNTU24-STIG-Audit` repo) to see how many controls passed and how many still fail. It's the gating test you run before merging or pushing benchmark changes.

End-to-end you get:

1. A clean container booted into systemd.
2. Pre-remediation audit (baseline): records which controls are already failing before the role runs.
3. Role converge #1: applies all the STIG remediations.
4. Role converge #2: re-runs to confirm idempotency (zero changed tasks the second time).
5. Post-remediation audit: records the after-picture.
6. Pre and post audit JSON summaries copied out beside the role for review.

If steps 3 and 4 both report `failed=0` and step 6 shows the post-audit failure count substantially lower than the pre-audit count, the role is shippable.

## Prerequisites

| Need | Why | How to check |
|---|---|---|
| Docker (Desktop or Engine), running | Molecule uses Docker to create the test container | `docker info` returns without error |
| Python 3.10+ | Ansible / Molecule are Python tools | `python3 --version` |
| Ansible venv with `ansible-core >= 2.16.1`, `molecule`, `molecule-plugins[docker]`, `docker`, `passlib` | Runtime deps for the test (matches `meta/main.yml` `min_ansible_version`) | `pip list \| grep -E 'ansible\|molecule'` |
| `git` on the controller | Audit content is cloned from `UBUNTU24-STIG-Audit` | `git --version` |
| Local clones of **both** this role AND `UBUNTU24-STIG-Audit` (or network access so the audit repo can be cloned at runtime) | The role pulls audit goss content from the audit repo during converge | `git -C <path-to>/UBUNTU24-STIG-Audit status` |

One-time venv setup (skip if you already have one):

```bash
python3 -m venv <path-to-your-ansible-venv>
source <path-to-your-ansible-venv>/bin/activate
pip install 'ansible-core>=2.16.1' 'molecule>=24' 'molecule-plugins[docker]' docker passlib
```

## Quick start

```bash
# Every time
source <path-to-your-ansible-venv>/bin/activate
cd <path-to-this-role>

# Full gating pair
molecule destroy && molecule converge && molecule converge && molecule verify
```

The whole sequence takes 5-15 minutes on a modern laptop. Network is needed (clone of audit content + apt installs) for the first run; subsequent runs reuse the cached Docker image.

## What each command does

| Command | What happens |
|---|---|
| `molecule destroy` | Removes any leftover container from a previous run. Safe to run on a clean system. |
| `molecule converge` (first) | (a) pulls / starts the `geerlingguy/docker-ubuntu2404-ansible:latest` container, (b) runs `prepare.yml` to install supporting packages, (c) clones the audit content from `UBUNTU24-STIG-Audit`, (d) runs the pre-remediation audit, (e) applies the full role, (f) runs the post-remediation audit, (g) fetches both audit JSONs out of the container for review. |
| `molecule converge` (second) | Re-runs the role against the already-remediated container. The Ansible `PLAY RECAP` should show `changed=0` (or a small handful of acceptable re-renders) - that's the **idempotency check**. |
| `molecule verify` | Final assertions defined in the scenario (light by default for this role; the real validation is in the audit JSONs). |

## Where the audit results land

After `molecule converge` completes, the pre/post audit JSONs are fetched out of the container (goss output). Each has a `results: []` array where every entry has a `successful: true|false` field. Compare pre vs. post:

```bash
# How many controls passed before remediation?
jq '[.results[] | select(.successful==true)] | length' <pre_scan>.json

# How many passed after?
jq '[.results[] | select(.successful==true)] | length' <post_scan>.json
```

If the second number is significantly higher than the first, the role is working as intended.

## How to read `PLAY RECAP`

After each converge, Ansible prints a one-line summary like:

```
PLAY RECAP ********************************************************
ubuntu2404 : ok=180  changed=60  unreachable=0  failed=0  skipped=120  rescued=0  ignored=0
```

| Counter | What it means |
|---|---|
| `ok` | Tasks that ran and finished successfully (no state change needed, or already in desired state) |
| `changed` | Tasks that modified the system (installed a package, edited a file, etc.) |
| `failed` | Tasks that errored out. **MUST BE 0** for the run to be considered passing. |
| `skipped` | Tasks gated off by `when:` conditions (often because the role detected a container and skipped container-incompatible work) |
| `rescued` / `ignored` | Block-level error handling - typically 0 |

**Idempotency expectation:** the second `converge` should show `changed=0` (or near zero) - the system was already in the desired state. A high `changed` count on the second run means a task is not idempotent and needs fixing. The pre/post audit shell tasks and the ansible-facts file task always re-render (they're actions, not state changes); 2-3 `changed` on the second converge driven by those is acceptable.

## Why some audit failures are expected (even after the role passes)

The post-scan will still show some controls failing. Most fall into three buckets that aren't role bugs - they're inherent to containerized testing:

1. **SSH-config controls** (banner, ciphers, KexAlgorithms, MACs, idle timeouts): on a fresh Ubuntu container `sshd` isn't running, so goss checks for active enforcement can't pass; on real hosts they pass after the `Restart_ssh` handler fires.
2. **User-state controls** (password complexity/history/lifetime, dotfile audit): need real interactive users (UID >= 1000) with shadow entries and home directories. Containers only have system accounts.
3. **Kernel / mount / boot controls**: any control gated by `not system_is_container` is intentionally skipped during remediation (auditd kernel rules, AppArmor profiles, kernel sysctls, mount options, GRUB / boot CMDLINE) because containers share the host kernel - see `vars/is_container.yml`. Audit still runs the goss tests, so they report as failing.

None of these warrant fixing the role. They're the expected delta between a containerized test and a real Ubuntu 24.04 host.

## Common gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `molecule converge` errors with "container not running" | Stale container from a previous interrupted run | Run `molecule destroy` first, then retry |
| Pre-task fails with "Unable to acquire the dpkg frontend lock" | Another apt process is running inside the prepared container | Re-run; if persistent, add a short sleep at the top of `prepare.yml` |
| `/bin/sh: 1: set: Illegal option -o pipefail` | A `shell:` task ran under dash (Ubuntu `/bin/sh` is dash, which has no `pipefail`) | Add `args: executable: /bin/bash` to that shell task |
| Converge #2 fails on a task that passed in #1 | Ansible 2.19+ struct-vs-string type-check tripping a `when:` clause that was "lucky" on the first run | Inspect the offending task's `when:` - look for quoted-string-as-boolean bugs |
| `Conditional result (True) was derived from value of type 'str'` | Ansible 2.19+ rejects when-clauses whose final value is a non-boolean string | Remove stray quotes; the value should be a bare Jinja expression |
| `verify` step empty / passes trivially | Expected - this role's primary validation is the goss audit JSONs, not molecule's `verifier:` plays | Inspect the audit JSONs instead |

## Apple Silicon (M-series Mac) note

`geerlingguy/docker-ubuntu2404-ansible:latest` is a multi-arch image (amd64 + arm64), so on Apple Silicon it pulls the arm64 variant natively - no Rosetta required. The scenario leaves `platform:` unspecified to take advantage of that.

If you specifically need to reproduce a CI runner's behaviour (which is amd64), add `platform: linux/amd64` to the platform entry in `molecule.yml`. That path requires Docker Desktop's "Use Rosetta for x86_64/amd64 emulation" toggle and runs noticeably slower.

## Reference: what's in the default scenario

| File | Purpose |
|---|---|
| `molecule.yml` | Driver: docker. Image: `geerlingguy/docker-ubuntu2404-ansible:latest` (multi-arch; native arm64 on Apple Silicon, amd64 elsewhere). systemd as PID 1 via `command: /lib/systemd/systemd`. Host vars: `ansible_become: false`, `setup_audit: true`, `run_audit: true`. |
| `prepare.yml` | Installs supporting packages (`openssh-server`, `libpam-pwquality`, `libpam-modules`, `sudo`, `acl`, `kmod`, `cron`, `chrony`, `rsyslog`, `aide`, `aide-common`, `logrotate`, `apparmor`, `apparmor-utils`, `ufw`, `nftables`, `auditd`, `autofs`, `python3-apt`). Creates privilege-separation and config directories. Starts `cron` and `rsyslog`. |
| `converge.yml` | Resolves the role by directory basename, sets a real root password (so PAM tasks don't lock us out), stubs `/etc/default/grub` so GRUB-tag tasks don't fail in the container, then includes the role. |

## Reference: key test-scaffolding vars (in `molecule/default/converge.yml`)

| Override | Value | Purpose |
|---|---|---|
| `ubtu24stig_skip_for_test` | `true` | Bypasses the role's "you must read the disclaimer" assertion that normally guards real-host runs. |
| `ubtu24stig_set_bootloader_password` | `false` | Bootloader assertions are skipped via container detection; this is belt-and-braces. |
| `change_requires_reboot` | `false` | Suppresses reboot handlers (containers can't reboot). |
| `ansible_become` | `false` | Container already runs as root; sudo via become is unnecessary. |
| `setup_audit` / `run_audit` | `true` | Drives the pre/post audit goss runs. |
| `system_is_container` | auto-detected | The role detects the container environment and gates container-incompatible tasks (auditd, kernel sysctls, mount changes, GRUB, AppArmor kernel-level rules) - see `vars/is_container.yml`. |

## Overriding the audit branch

`audit_git_version` defaults to `benchmark_{{ benchmark_version }}` (i.e. `benchmark_v1.5.0`) in `defaults/main/audit.yml`. If you're QA'ing a feature branch on `UBUNTU24-STIG-Audit` that hasn't been merged yet, override via extra-vars:

```bash
molecule converge -- --extra-vars 'audit_git_version=<your-qa-branch>'
```

`audit_git_version` is an ordinary role default in `defaults/main/audit.yml`, so inventory, group and play vars override it too - `--extra-vars` is simply the most direct option from the molecule command line.

**Branch naming reminder:** the remediation repo uses DISA-style `benchmark_v1rN` (no dots), but the audit repo uses semver `benchmark_v1.N.0`. Always pass the full audit branch name to `audit_git_version` (e.g. `benchmark_v1.5.0`, not `benchmark_v1r5`).

## Where to ask for help

- This role's open issues: `github.com/ansible-lockdown/UBUNTU24-STIG/issues`
- Ansible Lockdown Discord (linked from the role README badge)
- Molecule docs: `molecule.readthedocs.io`
- Goss output format: `github.com/goss-org/goss`
