# Branch protection & approval setup

This is what turns the monorepo from "a folder of YAML" into a governed platform.
Configure these once on the `main` branch.

## GitHub

**Settings → Branches → Add branch ruleset (or classic branch protection) for `main`:**

- [x] Require a pull request before merging
  - [x] Require approvals — set to **1** (or 2 for production-critical domains)
  - [x] **Require review from Code Owners**  ← this is the key setting; it makes
        CODEOWNERS binding, so a PR touching `network/` cannot merge without a
        Network-team approval.
  - [x] Dismiss stale approvals when new commits are pushed
- [x] Require status checks to pass before merging
  - Select: `lint-all`, and the per-domain jobs (`cloud`, `vm`, `storage`,
    `network`, `servers`). Path-filtered jobs that don't run are skipped, not failed.
- [x] Require branches to be up to date before merging
- [x] Do not allow bypassing the above settings (applies the rule to admins too)
- [x] Restrict who can push to matching branches → allow no direct pushes

**Teams (Settings → Teams) referenced by CODEOWNERS:**
`@org/platform-admins`, `@org/team-cloud`, `@org/team-virtualization`,
`@org/team-storage`, `@org/team-network`, `@org/team-windows`, `@org/team-linux`.

## GitLab

GitLab's equivalent of CODEOWNERS is the same file at `.gitlab/CODEOWNERS`,
`CODEOWNERS`, or `docs/CODEOWNERS`. Then:

- **Settings → Repository → Protected branches:** protect `main`, set
  "Allowed to merge" to Maintainers, "Allowed to push" to No one.
- **Settings → Merge requests:** enable "All threads must be resolved" and
  "Require approval from code owners."
- Add Code Owner approval as a required approval rule for `main`.

## The takeaway for your customer

"A few admins approve everything" is replaced by "the right team approves the
changes they understand, automatically." Platform admins govern the shared rails;
domain experts govern their domain. Nothing reaches `main` unreviewed, and nothing
waits on someone who lacks the context to review it.
