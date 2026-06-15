# AAP Automation Platform — Enterprise Monorepo

A single Git repository holding all Ansible Automation Platform (AAP) content for
the enterprise, organized by infrastructure domain. One repo, one SDLC, one set of
CI/CD rules — governed so that each domain team owns and approves its own changes
without funneling every pull request through a small group of platform admins.

## Why one repo (and how it stays governed)

A common mistake when consolidating into a monorepo is to make a handful of
platform admins the only people who can approve pull requests. That creates a
bottleneck and puts approval in the hands of people who may not understand the
change (a storage admin approving a DNS edit). This repo avoids that with two
GitHub features working together:

| Mechanism | What it does |
|-----------|--------------|
| `.github/CODEOWNERS` | Routes each PR to the **owning team** for the folder it touches. A change under `network/` requires Network-team approval; under `storage/`, the Storage team; and so on. |
| Branch protection on `main` | Requires a passing CI pipeline **and** "Review from Code Owners" before merge. Nobody — including admins — self-merges to `main`. |

The result: **distributed approval, central standards.** Domain teams approve their
own domain's PRs. Platform admins own only the shared machinery (`/.github/`,
`/collections/`, `/execution-environment/`, `/common/`, `/eda/`).

See [`docs/branch-protection.md`](docs/branch-protection.md) for the exact settings.

## Repository layout

```
aap-automation-platform/
├── .github/
│   ├── CODEOWNERS              # per-folder review routing (the governance core)
│   └── workflows/ci.yml        # GitHub Actions: lint repo-wide, Molecule per changed domain
├── .gitlab-ci.yml              # GitLab CI equivalent (same path-based rules)
├── ansible.cfg                 # roles_path spans every domain's roles/
├── collections/requirements.yml
├── execution-environment/      # ansible-builder EE definition (shared)
├── common/
│   └── roles/servicenow_writeback/   # shared result write-back, reused by all domains
├── cloud/                      # Cloud team  — AWS / Azure VM provisioning
│   ├── roles/provision_aws_vm/
│   └── playbooks/provision_vm.yml
├── vm/                         # Virtualization team — Nutanix VM provisioning
│   ├── roles/provision_nutanix_vm/
│   └── playbooks/provision_nutanix_vm.yml
├── storage/                    # Storage team — NetApp ONTAP volume
│   ├── roles/provision_ontap_volume/
│   └── playbooks/provision_volume.yml
├── network/                    # Network team — Infoblox DNS record
│   ├── roles/manage_dns_record/
│   └── playbooks/manage_dns_record.yml
├── servers/                    # Windows + Linux teams — Day-2 baseline
│   ├── roles/configure_baseline/
│   └── playbooks/configure_baseline.yml
├── eda/
│   └── rulebooks/drift_remediation.yml
└── docs/
```

## The SDLC, in one loop

Every domain follows the identical flow:

1. Engineer works on a `feature/*` branch in their domain folder.
2. Opens a PR → CI runs `yamllint` + `ansible-lint` repo-wide, and `molecule test`
   for **only the domain(s) that changed** (path-filtered, so the repo stays fast).
3. CODEOWNERS requests the owning team's review; branch protection blocks merge
   until it's approved and CI is green.
4. Merge to `main` → the AAP Project syncs from SCM.
5. Promotion Dev → Test → Prod via separate job templates tracking branches/tags.
6. ServiceNow catalog request → approval → AAP job template → result written back
   to the ticket by the shared `servicenow_writeback` role.

## Mapping the monorepo to AAP

You have two clean options for wiring this single repo into Automation Controller:

- **One Project, many job templates (simplest).** Create one Controller *Project*
  pointed at this repo. Create a job template per playbook, each with its
  `playbook` field set to the domain path (e.g. `network/playbooks/manage_dns_record.yml`).
- **One Project per domain, scoped by organization (stronger RBAC).** Create a
  Project per domain all pointing at the same repo URL, and assign each to the
  matching Controller *Organization* (Cloud, Network, Storage, …). This lets
  Controller RBAC mirror the CODEOWNERS model: the Network org's users only see
  Network job templates.

Either way, the **execution environment** is shared and built from
`execution-environment/execution-environment.yml`.

## Local quick start

```bash
# Install collections into the repo-local path
ansible-galaxy collection install -r collections/requirements.yml

# Lint everything
yamllint .
ansible-lint

# Test one domain's role
cd network/roles/manage_dns_record && molecule test
```

New to the repo? Read **[`CONTRIBUTING.md`](CONTRIBUTING.md)** — it walks a domain
engineer through the full branch → PR → review → merge flow.

The **network DNS role** is the fully-wired reference for testing: see
[`network/roles/manage_dns_record/molecule/default/README.md`](network/roles/manage_dns_record/molecule/default/README.md)
for the pattern (a `dns_check_mode` gate lets the role's own logic be tested in CI
with no live infrastructure). Copy that pattern to flesh out the other domains'
placeholder scenarios.

## Notes on portability (no lock-in)

Everything here is plain YAML running on `ansible-core`. `yamllint`, `ansible-lint`,
`molecule`, and `ansible-builder` are upstream open-source tools. The collections are
open source and published on Ansible Galaxy as well as in the certified channel. The
AAP subscription adds the supported Controller, certified content, EEs, EDA, RBAC,
and support — not a proprietary language you can't leave.
