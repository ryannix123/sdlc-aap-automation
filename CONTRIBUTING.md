# Contributing — the domain engineer's workflow

This guide walks a domain engineer (cloud, network, storage, virtualization, or
server team) through making a change to this monorepo, from a fresh clone to a
merged, promotable pull request. The flow is identical for every domain; only the
folder you work in changes.

The goal is simple: **every change is reviewed by the team that owns it before it
can run in the enterprise.** You don't need to know the whole repo — just your
domain folder and these steps.

---

## 1. One-time setup

```bash
git clone git@github.com:org/aap-automation-platform.git
cd aap-automation-platform

# Install the collections into the repo-local path (matches CI and the EE)
ansible-galaxy collection install -r collections/requirements.yml

# Confirm your tooling is present (all upstream open source)
ansible-lint --version
yamllint --version
molecule --version
```

You only ever edit files inside your domain's folder (e.g. `network/`). The shared
folders (`collections/`, `common/`, `.github/`, `eda/`) are owned by the platform
admins; if your change needs something there, flag it in your PR description.

---

## 2. Start a feature branch

One logical change per branch. Name it so reviewers know the domain and intent.

```bash
git switch main
git pull --ff-only
git switch -c feature/network-add-cname-support
```

---

## 3. Make the change in your domain folder

Edit the role under `<domain>/roles/<role>/`. Roles are the reusable unit;
playbooks under `<domain>/playbooks/` stay thin and just call the role plus the
shared write-back. For example, a network engineer adding CNAME support edits:

```
network/roles/manage_dns_record/tasks/main.yml
network/roles/manage_dns_record/defaults/main.yml   # new variable defaults
network/roles/manage_dns_record/molecule/default/   # extend the test
```

If you add a new variable, give it a sensible default in `defaults/main.yml` and
document it in the role's task comments so the next person — and the reviewer —
understands it.

---

## 4. Validate locally before you push

Run the same checks CI will run. Fixing them now is faster than waiting for the
pipeline.

```bash
# Format / style
yamllint .

# Ansible best-practice and idempotency checks
ansible-lint

# Test your role in isolation: converge, prove idempotency, verify, destroy
cd network/roles/manage_dns_record
molecule test
cd -
```

`molecule test` runs the full sequence. While iterating, you can run just part of
it to save time:

```bash
molecule converge   # apply the role once
molecule verify     # run the assertions
molecule destroy    # clean up
```

A green `molecule test` is the bar. If it passes locally, it will pass in CI.

---

## 5. Open the pull request

```bash
git add network/
git commit -m "network: add CNAME record support to manage_dns_record"
git push -u origin feature/network-add-cname-support
```

Open the PR against `main`. The moment you do:

- **CI runs automatically.** `yamllint` and `ansible-lint` run repo-wide; Molecule
  runs for **only the domain(s) you touched** (path-filtered, so a network PR
  doesn't run the cloud tests).
- **CODEOWNERS routes the review.** Because you changed files under `network/`,
  GitHub automatically requests a review from `@org/team-network`. You don't pick
  reviewers — ownership does.

Write a PR description that helps your reviewer: what changed, why, any new
variables, and how you tested it.

---

## 6. Review and merge

- A member of the **owning team** reviews. They're accountable for what runs in
  their domain, so they review with that context.
- Branch protection blocks merge until **both** conditions are met: CI is green
  **and** a Code Owner has approved. Nobody — including admins — can self-merge to
  `main`. (See `docs/branch-protection.md` for the exact settings.)
- Address review comments by pushing more commits to the same branch; CI re-runs
  and stale approvals are dismissed so the latest code is what gets approved.
- Once approved and green, merge (squash is recommended to keep `main` history
  clean).

---

## 7. What happens after merge

You don't deploy by merging — merging makes the content *available*. Promotion is
a separate, deliberate act:

1. The AAP Project syncs from `main` automatically on merge.
2. Your role/playbook is now runnable by its job template in **Dev**.
3. Promotion to **Test** then **Prod** happens by the platform/change process
   pointing those job templates at a reviewed branch or tag — an explicit, logged
   step, not a side effect of your merge.

---

## Quick reference

| You want to… | Do this |
|--------------|---------|
| Start work | `git switch -c feature/<domain>-<short-desc>` |
| Check style | `yamllint .` |
| Check best practices | `ansible-lint` |
| Test your role | `cd <domain>/roles/<role> && molecule test` |
| Get it reviewed | Open a PR — CODEOWNERS requests your team automatically |
| Know who approves | The team that owns the folder you changed |
| Ship to Prod | Merge → then promote via job template (platform/change process) |

## Etiquette

- One logical change per PR — small PRs review faster and fail less.
- Never edit another domain's folder in the same PR; split it.
- If you genuinely need a change in a shared folder (`collections/`, `common/`),
  call it out clearly so the platform admins reviewing that part know to look.
- Keep playbooks thin; put real logic in roles so it's testable with Molecule.
