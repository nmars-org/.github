# Org SECURITY.md Auditor

`monitor-security.yaml` audits every public, non-archived repository in the
organization for a `SECURITY.md` on its default branch and publishes a
consolidated report to the workflow run summary.

Runs nightly at 00:00 UTC, and on demand via **Run workflow**.

## Why

The EU Cyber Resilience Act (CRA) requires all in-scope repositories to have a
SECURITY.md file documenting vulnerability reporting procedures, steward contact information, and Coordinated Vulnerability Disclosure (CVD) process.

More details about CRA stewardship can be found here:
[http://red.ht/cra-steward](http://red.ht/cra-steward)

The canonical SECURITY.md lives in one place — the org's `.github`
repository — and each repo's file is a short stub linking back to it, so the
policy itself stays centrally maintained and cannot drift copy by copy.

This workflow is the detective control for that: it reports which repos are
missing the file. It is reporting only — it makes no changes to any repository,
and remediation is left to a human.

## How it works

Three jobs.

### `audit`

A single paginated GraphQL query returns every public, non-archived repo along
with the **text** of its `SECURITY.md`, if any:

```graphql
securityMd: object(expression: "HEAD:SECURITY.md") {
  ... on Blob { text }
}
```

Pulling the body inline is what makes the content check free — no extra request
per repo. At 134 repos it adds ~24 KB to the response.

Each repo is then classified into one of three states:

| Status | Meaning |
| :--- | :--- |
| `ok` | `SECURITY.md` exists **and** its body matches `POLICY_LINK_PATTERN` |
| `unlinked` | `SECURITY.md` exists but never references the canonical policy |
| `missing` | No `SECURITY.md` on the default branch |

The body is tested and then **discarded** — it must not reach the job output,
which is capped at 1 MB. Only the resulting status travels onward.

The job subtracts the exemption list and emits the results as JSON.

#### Matching the link

`POLICY_LINK_PATTERN` is an extended regex matched case-insensitively:

```yaml
POLICY_LINK_PATTERN: 'github\.com/konflux-ci/\.github'
```

It is deliberately **not** an exact comparison against `POLICY_URL`. Real stubs
link via `/blob/main/`, `/blob/HEAD/`, `/security/policy`, or
`?tab=security-ov-file`, and all of those are correct. In konflux-ci every
existing stub uses `/blob/main/` while `POLICY_URL` is written `/blob/HEAD/`, so
a literal match would have flagged all 91 conforming repos as broken.

The trade-off is that the check verifies the file *points at* the canonical
policy — not that it says anything sensible around the link.

### `report`

Renders the Markdown run summary and emits a `::warning` annotation per
non-compliant repo. Guarded with `if: always()` so a failed audit still explains
itself on the run page instead of leaving it blank. It does **not** fail on
non-compliance — rendering the report succeeded either way.

### `compliance-gate`

Holds the pass/fail decision, and nothing else. Fails when any in-scope repo is
missing `SECURITY.md`. See [Reading the run status](#reading-the-run-status).

## What the report shows

- Compliance counts and a 20-cell coverage bar
- A table of repos missing `SECURITY.md`, with their default branch
- A separate table of repos whose `SECURITY.md` does not link to the canonical
  policy, each linked to the offending file
- Collapsible lists of compliant repos and of exemptions with their reasons
- A `::warning` annotation per non-compliant repo
- A failing exit if anything is missing

## Reading the run status

GitHub Actions has **no amber "warning" conclusion** for a run, job, or step. A
run is green, red, or grey. `continue-on-error: true` does not help — the step's
`outcome` becomes `failure` but its `conclusion` becomes `success`, so it renders
green. The Checks API supports a `neutral` conclusion, but the runner cannot emit
it for its own jobs. There is an open feature request for a `warn-on-failure`
flag; it does not exist today.

So the two failure modes are separated by **job** instead of by colour. Read the
job list, not the run badge:

| Job list | Meaning |
| :--- | :--- |
| ❌ `Audit SECURITY.md` | The workflow itself is broken — bad token, GraphQL failure, malformed exemption file, shell error. `Report Results` goes red explaining itself; `Compliance Gate` is **skipped**. |
| ✅ audit, ✅ report, ❌ `Compliance Gate` | The workflow ran clean. Repos are non-conforming. This is the expected steady state until coverage hits 100%. |
| ✅ all three | Every in-scope repo has a `SECURITY.md`. |

`compliance-gate` carries no `if:` on purpose. The default `success()` condition
means it is skipped rather than run whenever `audit` or `report` failed, so a
broken workflow never also trips the compliance gate — the two signals stay
independent.

To run this as a pure dashboard, delete the `compliance-gate` job. The report and
its warning annotations are unaffected.

## Exemptions

Listed in `.github/security-md-exemptions.yaml` as `repo` / `reason` pairs:

```yaml
exemptions:
  - repo: brand-assets
    reason: "No product code"
```

Kept in the repository rather than in an Actions variable so that changes go
through review, carry a reason, and leave a git history of who exempted what
and when. Matching is on the repository name, since the auditor only ever scans
one organization.

Entries that no longer match a repo in scope emit a **stale exemption** warning
rather than failing silently, so the list does not rot as repos are renamed,
archived, or deleted.

The list is edited by hand. The workflow only reads it.

## Setup

1. Set `ORG_NAME`, `POLICY_URL`, and `POLICY_LINK_PATTERN` in the workflow's
   `env:` block. The pattern must match however your stubs actually link back —
   check a few real files before trusting it.
2. Add `ORG_MONITOR_TOKEN` as a repository secret. **Read access only** — the
   workflow never writes to any repository, and no write scope is required.
3. Commit `.github/security-md-exemptions.yaml` alongside the workflow. It is
   read via `actions/checkout`, so it must live in the same repository; a
   missing file fails the audit step.
4. Trigger once manually and review the summary before letting the nightly
   schedule take over.

## Behavior notes

- **The run fails when any in-scope repo is missing `SECURITY.md`**, via the
  `compliance-gate` job. Expect the first run to be red, and see
  [Reading the run status](#reading-the-run-status) for telling that apart from
  a genuine workflow error.
- **Repos with zero commits** have no default branch, so they cannot hold a
  `SECURITY.md`. They are flagged as `_none — empty repo, no commits_` in the
  branch column. Consider exempting them outright.
- **Archived and private repos are out of scope** by construction, filtered in
  the GraphQL query rather than after the fact.
- **A hand-written policy counts as non-conforming.** A repo with its own
  substantive `SECURITY.md` that never links to the canonical one is reported as
  `unlinked` and fails the gate, because the stated goal is a single centrally
  maintained policy. If you would rather treat that as acceptable, drop
  `UNLINKED` from the `NONCONFORMING` sum in `compliance-gate` — it then stays
  in the report as advisory only.
- **Root only.** The audit reads `SECURITY.md` at the repository root. GitHub
  also honours `.github/SECURITY.md` and `docs/SECURITY.md`; neither is used
  anywhere in konflux-ci (verified 2026-09-23), so the root-only check is not
  currently missing anything. Add more `object(expression:)` aliases if that
  changes.
- **New repos are the main source of drift.** Compliance correlates strongly
  with repo age — recently created repos are the least likely to have the file.
  A nightly audit will keep finding new gaps; seeding the stub at repo-creation
  time is the durable fix, with this workflow as the safety net.
