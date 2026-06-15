# Molecule scenario — manage_dns_record

This is the fully-wired reference test for the monorepo. The other domains'
scenarios are placeholders; copy this pattern to flesh them out.

## What it proves

`molecule test` runs the full sequence — `converge → idempotence → verify` —
against `localhost`, with no container, cloud, or live Infoblox grid required:

- **converge.yml** runs the role twice (once as an A record, once as a CNAME)
  with `dns_check_mode: true`. That flag skips the live grid API call and
  produces a simulated result instead, so the role's *own* logic — input
  validation, branch selection (A vs CNAME), and the write-back facts the
  ServiceNow role consumes — is exercised end to end.
- **idempotence** confirms a second run reports no changes.
- **verify.yml** asserts the role set the correct `provisioned_name`,
  `provisioned_ip`, and `dns_record_ref` for both record types.

Captured facts are persisted across the converge→verify play boundary via a
`jsonfile` fact cache configured in `molecule.yml`.

## How the gate works

The role (`tasks/main.yml`) wraps the real `infoblox.nios_modules` calls in a
`when: not dns_check_mode` block. In production `dns_check_mode` defaults to
`false`, so the live calls run. In CI it's `true`. This is an honest split: only
the external grid call is simulated; everything the role is responsible for is
genuinely tested.

## Running locally

```bash
# One-time: install collections (needs galaxy / certified-content access)
ansible-galaxy collection install -r ../../../../collections/requirements.yml

cd network/roles/manage_dns_record
molecule test          # full sequence
# or, while iterating:
molecule converge
molecule verify
molecule destroy
```

In CI this runs inside `quay.io/ansible/creator-ee`, where the collection is
present and Molecule's dependency step installs `requirements.yml`.
