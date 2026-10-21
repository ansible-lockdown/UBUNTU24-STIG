# Molecule Quick Start - Ubuntu 24.04 LTS STIG

> Audience: anyone new to molecule testing with this Ansible Lockdown role. Read top-to-bottom the first time; come back to the reference tables later.

## What this scenario does

This `molecule/` directory ships one scenario: `default/`. It spins up a throwaway Ubuntu 24.04 LTS Docker container, applies the **whole** Ansible Lockdown Ubuntu 24.04 STIG role against it, then runs the paired audit with syver (content from the `UBUNTU24-STIG-Audit` repo) to see how many controls passed and how many still fail. It's the gating test you run before merging or pushing benchmark changes.

End-to-end you get:

1. A clean container booted into systemd.
2. Pre-remediation audit (baseline): records which controls are already failing before the role runs.
3. Role converge #1: applies all the STIG remediations.
4. Idempotence: `molecule idempotence` re-runs the role and fails on any changed task.
5. Post-remediation audit: records the after-picture.
6. Pre and post audit JSON summaries copied out beside the role for review.

If step 3 reports `failed=0`, step 4 reports `changed=0`, and step 6 shows the post-audit failure count substantially lower than the pre-audit count, the role is shippable.

## Prerequisites

| Need | Why | How to check |
|---|---|---|
| Docker (Desktop or Engine), running | Molecule uses Docker to create the test container | `docker info` returns without error |
| Python 3.10+ | Ansible / Molecule are Python tools | `python3 --version` |
| Ansible venv with `ansible-core >= 2.16.1`, `molecule`, `molecule-plugins[docker]`, `docker`, `passlib` | Runtime deps for the test (matches `meta/main.yml` `min_ansible_version`) | `pip list \| grep -E 'ansible\|molecule'` |
| `git` on the controller | Audit content is cloned from `UBUNTU24-STIG-Audit` | `git --version` |
| Local clones of **both** this role AND `UBUNTU24-STIG-Audit` (or network access so the audit repo can be cloned at runtime) | The role pulls audit content from the audit repo during converge | `git -C <path-to>/UBUNTU24-STIG-Audit status` |

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

# Full gating sequence, as CI runs it
molecule destroy && molecule create && molecule converge && molecule idempotence && molecule verify
```

The whole sequence takes 5-15 minutes on a modern laptop. Network is needed (clone of audit content + apt installs) for the first run; subsequent runs reuse the cached Docker image.

## What each command does

| Command | What happens |
|---|---|
| `molecule destroy` | Removes any leftover container from a previous run. Safe to run on a clean system. |
| `molecule converge` (first) | (a) pulls / starts the `geerlingguy/docker-ubuntu2404-ansible:latest` container, (b) runs `prepare.yml` to install supporting packages, (c) clones the audit content from `UBUNTU24-STIG-Audit`, (d) runs the pre-remediation audit, (e) applies the full role, (f) runs the post-remediation audit, (g) fetches both audit JSONs out of the container for review. |
| `molecule idempotence` | Re-runs the role against the already-remediated container, skipping tasks tagged `molecule-idempotence-notest` (the audit scans, the audit fetch and the two facts templates, which write a timestamp or a fresh result every run). Any other changed task fails the step. |
| `molecule verify` | Runs the 25 tasks in `verify.yml`, a narrow regression check on top of converge that includes a lockout test for a throwaway user; the broad picture is in the audit JSONs. |

## Where the audit results land

After `molecule converge` completes, the pre/post audit JSONs are fetched out of the container (syver JSON output). Each has a `results: []` array where every entry has a `successful: true|false` field. Compare pre vs. post:

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

**Idempotence expectation:** `molecule idempotence` must show `changed=0`. The audit scans, the audit fetch and the facts templates always re-run by design, so they carry the `molecule-idempotence-notest` tag and are skipped by that step. Anything else that reports changed is a task that is not idempotent and needs fixing.

## Why some audit failures are expected (even after the role passes)

The post-scan will still show some controls failing. Most fall into these buckets that aren't role bugs - they're inherent to containerized testing or to the role's defaults:

1. **`ubtu24stig_disruption_high` controls**: the default is `false`, so the controls gated on it are reported but not remediated.
2. **User-state controls** (password complexity/history/lifetime, dotfile audit): need real interactive users (UID >= 1000) with shadow entries and home directories. Containers only have system accounts.
3. **Kernel / mount / boot controls**: any control gated by `not system_is_container` is intentionally skipped during remediation (auditd kernel rules, AppArmor profiles, kernel sysctls, mount options, GRUB / boot CMDLINE) because containers share the host kernel - see `vars/is_container.yml`. Audit still runs those tests, so they report as failing.

None of these warrant fixing the role. They're the expected delta between a containerized test and a real Ubuntu 24.04 host.

## Common gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `molecule converge` errors with "container not running" | Stale container from a previous interrupted run | Run `molecule destroy` first, then retry |
| Pre-task fails with "Unable to acquire the dpkg frontend lock" | Another apt process is running inside the prepared container | Re-run; if persistent, add a short sleep at the top of `prepare.yml` |
| `/bin/sh: 1: set: Illegal option -o pipefail` | A `shell:` task ran under dash (Ubuntu `/bin/sh` is dash, which has no `pipefail`) | Add `args: executable: "{{ ubtu24stig_shell_executable }}"` to that shell task (`/bin/bash` by default) |
| Converge #2 fails on a task that passed in #1 | Ansible 2.19+ struct-vs-string type-check tripping a `when:` clause that was "lucky" on the first run | Inspect the offending task's `when:` - look for quoted-string-as-boolean bugs |
| `Conditional result (True) was derived from value of type 'str'` | Ansible 2.19+ rejects when-clauses whose final value is a non-boolean string | Remove stray quotes; the value should be a bare Jinja expression |
| `molecule idempotence` fails on a task | That task changes something on every run | Make the task idempotent; tag it `molecule-idempotence-notest` only if it reports state by design, as the audit tasks do |

## Apple Silicon (M-series Mac) note

`geerlingguy/docker-ubuntu2404-ansible:latest` is a multi-arch image (amd64 + arm64). `molecule.yml` sets `platform: "${MOLECULE_DOCKER_PLATFORM:-linux/arm64}"`, so on Apple Silicon it runs the arm64 variant natively - no Rosetta required.

CI exports `MOLECULE_DOCKER_PLATFORM=linux/amd64`, because the CI runners are amd64. Export the same to reproduce it locally; on Apple Silicon that path requires Docker Desktop's "Use Rosetta for x86_64/amd64 emulation" toggle and runs noticeably slower. On an amd64 Linux host, export it to run natively.

## Reference: what's in the default scenario

| File | Purpose |
|---|---|
| `molecule.yml` | Driver: docker. Image: `geerlingguy/docker-ubuntu2404-ansible:latest` (multi-arch; native arm64 on Apple Silicon, amd64 elsewhere). systemd as PID 1 via `command: /lib/systemd/systemd`. Host vars: `ansible_become: false`, `setup_audit: true`, `run_audit: true`, `fetch_audit_output: true`, `audit_output_destination`. |
| `prepare.yml` | Installs supporting packages (`openssh-server`, `libpam-pwquality`, `libpam-modules`, `sudo`, `acl`, `kmod`, `cron`, `chrony`, `rsyslog`, `aide`, `aide-common`, `logrotate`, `apparmor`, `apparmor-utils`, `ufw`, `nftables`, `auditd`, `autofs`, `python3-apt`). Creates privilege-separation and config directories. Starts `cron` and `rsyslog`. |
| `converge.yml` | Resolves the role by directory basename, sets a real root password (so PAM tasks don't lock us out), stubs `/etc/default/grub` so GRUB-tag tasks don't fail in the container, then includes the role. |

## Reference: key test-scaffolding vars

| Override | Where | Value | Purpose |
|---|---|---|---|
| `ubtu24stig_bootloader_password_hash` | `converge.yml` | dummy hash | The bootloader assertion runs only on non-container hosts; a non-default value keeps any stray check passing. |
| `skip_reboot` | `converge.yml` | `true` | Containers can't reboot. This is also the role default. |
| `ansible_become` | `molecule.yml` | `false` | Container already runs as root; sudo via become is unnecessary. |
| `setup_audit` / `run_audit` / `fetch_audit_output` | `molecule.yml` | `true` | Drives the pre/post audit runs and fetches their output. |
| `system_is_container` | auto-detected | The role detects the container environment and gates container-incompatible tasks (auditd, kernel sysctls, mount changes, GRUB, AppArmor kernel-level rules) - see `vars/is_container.yml`. |

## Overriding the audit branch

`audit_git_version` defaults to `benchmark_{{ benchmark_version }}` (i.e. `benchmark_v1.6.0`) in `defaults/main/audit.yml`. If you're QA'ing a feature branch on `UBUNTU24-STIG-Audit` that hasn't been merged yet, override via extra-vars:

```bash
molecule converge -- --extra-vars 'audit_git_version=<your-qa-branch>'
```

`audit_git_version` is an ordinary role default in `defaults/main/audit.yml`, so inventory, group and play vars override it too - `--extra-vars` is simply the most direct option from the molecule command line.

**Branch naming reminder:** the remediation repo uses DISA-style `benchmark_v1rN` (no dots), but the audit repo uses semver `benchmark_v1.N.0`. Always pass the full audit branch name to `audit_git_version` (e.g. `benchmark_v1.6.0`, not `benchmark_v1r6`).

## Where to ask for help

- This role's open issues: `github.com/ansible-lockdown/UBUNTU24-STIG/issues`
- Ansible Lockdown Discord (linked from the role README badge)
- Molecule docs: `molecule.readthedocs.io`
- Audit binary (syver): `github.com/krameff/syver`
