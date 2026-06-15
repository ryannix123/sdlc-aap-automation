# SDLC — Virtualization team

This is the step-by-step contribution guide for the `vm/` domain. It mirrors
the repo-wide [`CONTRIBUTING.md`](../CONTRIBUTING.md) but uses this domain's role,
playbook, and collection so you can copy/paste.

**You own this folder.** Pull requests touching `vm/` are routed to
`@org/team-virtualization` automatically by [`.github/CODEOWNERS`](../.github/CODEOWNERS), and
branch protection blocks merge until a member of that team approves and CI is green.

---

## What's in here

| Path | Purpose |
|------|---------|
| `vm/roles/provision_nutanix_vm/` | The reusable role (tasks, defaults, meta, molecule scenario) |
| `vm/playbooks/provision_nutanix_vm.yml` | Thin wrapper launched by the AAP job template |
| Primary collection | `nutanix.ncp` |

The playbook stays thin — it calls the role, then the shared
[`servicenow_writeback`](../common/roles/servicenow_writeback/) role. Put real
logic in the role so it is testable with Molecule.

---

## 1. Set up once

```bash
git clone git@github.com:org/aap-automation-platform.git
cd aap-automation-platform
ansible-galaxy collection install -r collections/requirements.yml
```

Confirm your tooling (all upstream open source):

```bash
ansible-lint --version
yamllint --version
molecule --version
```

## 2. Start a feature branch

One logical change per branch. Name it so reviewers see the domain and intent.

```bash
git switch main
git pull --ff-only
git switch -c feature/vm-add-disk-tier-option
```

## 3. Make the change

Edit the role under `vm/roles/provision_nutanix_vm/`:

```
vm/roles/provision_nutanix_vm/tasks/main.yml        # the logic
vm/roles/provision_nutanix_vm/defaults/main.yml     # variable defaults
vm/roles/provision_nutanix_vm/molecule/default/     # the test that proves it works
```

New variable? Give it a default in `defaults/main.yml` and a one-line comment in
`tasks/main.yml` so the reviewer understands it.

> **Note on testing without live infrastructure.** This role gates its live
> Virtualization team calls behind `vm_check_mode`. In production that defaults to `false` so the
> real calls run; in CI Molecule sets it `true`, which exercises the role's own
> logic (validation, branch selection, the write-back facts) without touching real
> systems. When you add behavior, extend the converge/verify scenario to cover it.

## 4. Validate locally (run what CI runs)

```bash
# Style + best practices (run from repo root)
yamllint .
ansible-lint

# Test this role in isolation: converge, prove idempotency, verify, destroy
cd vm/roles/provision_nutanix_vm
molecule test
cd -
```

While iterating you can run parts of the sequence:

```bash
molecule converge   # apply the role once
molecule verify     # run the assertions
molecule destroy    # clean up
```

A green `molecule test` is the bar. If it passes locally, it passes in CI.

## 5. Open the pull request

```bash
git add vm/
git commit -m "vm: <short description of the change>"
git push -u origin feature/vm-add-disk-tier-option
```

Open the PR against `main`. The moment you do:

- **CI runs automatically** — `yamllint` + `ansible-lint` repo-wide, and
  `molecule test` for the `vm` job (path-filtered; other domains don't run).
- **CODEOWNERS requests `@org/team-virtualization`** because you changed files under `vm/`.
  You don't pick reviewers — ownership does.

Write a description that helps the reviewer: what changed, why, any new variables,
and how you tested it.

## 6. Review & merge

- A member of **Virtualization team** reviews — they're accountable for what runs in this domain.
- Branch protection blocks merge until CI is green **and** a Code Owner approves.
  Nobody self-merges to `main`.
- Push more commits to address feedback; CI re-runs and stale approvals are
  dismissed so the latest code is what ships.
- Once approved and green, merge (squash recommended).

## 7. After merge

Merging makes the content *available*; it does not deploy it.

1. The AAP Project syncs from `main`.
2. Your role/playbook is runnable by its job template in **Dev**.
3. Promotion to **Test** then **Prod** is an explicit, logged step via the job
   template pointing at a reviewed branch or tag — not a side effect of merge.

---

## Quick reference

| You want to… | Do this |
|--------------|---------|
| Start work | `git switch -c feature/vm-add-disk-tier-option` |
| Check style | `yamllint .` |
| Check best practices | `ansible-lint` |
| Test this role | `cd vm/roles/provision_nutanix_vm && molecule test` |
| Get it reviewed | Open a PR — `@org/team-virtualization` is requested automatically |
| Ship to Prod | Merge → then promote via the job template |
