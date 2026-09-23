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
missing the file and gives maintainers a one-click path to add it or add the
repository to an exemption list.

## How it works

Two jobs.

### `audit`

A single paginated GraphQL query returns every public, non-archived repo along
with whether `SECURITY.md` exists on its default branch:

```graphql
securityMd: object(expression: "HEAD:SECURITY.md") { __typename }
```

The job then subtracts the exemption list and emits the results as JSON.

### `report`

Renders the Markdown run summary. Guarded with `if: always()` so a failed audit
still explains itself on the run page instead of leaving it blank.

## What the report shows

- Compliance counts and a 20-cell coverage bar
- A linked table of non-compliant repos, each with a remediation link
- Collapsible lists of compliant repos and of exemptions with their reasons
- A `::warning` annotation per non-compliant repo
- A failing exit if anything is missing

## Remediation links

Every non-compliant repo gets two one-click links, reflecting the two valid
responses to a missing policy — add one, or declare the repo out of scope.

| Link | Opens | Result |
| :--- | :--- | :--- |
| **🔀 Open PR** | the target repo's new-file editor, stub prefilled | adds `SECURITY.md` to that repo |
| **🛡️ Exempt it** | this repo's exemption file, entry already appended | adds the repo to the exemption list |

Both carry `quick_pull=1`, which preselects _"Create a new branch for this
commit and start a pull request"_. A link cannot create a commit, and a pull
request cannot exist without one, so the maintainer still clicks **Commit
changes** — but the default path is a pull request rather than a direct write
to the default branch.

That matters most on repos **without** branch protection. Protected repos force
the pull request path regardless; unprotected ones would otherwise accept a
direct commit to the default branch straight from the editor. The parameter
sets the default selection, it does not lock it, so the summary carries a note
telling reviewers not to switch the dialog back.

The **🛡️ Exempt it** link prefills the entry with a `TODO` reason, so a
reviewer cannot merge a silent exemption without writing a justification.

> `quick_pull=1` and `value=` are lightly documented. Verify they behave as
> described in your environment before relying on them. If they do not, the
> links still open the correct file in the correct editor, and the dialog still
> offers the pull request option — it just will not be preselected.

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

The report's **🛡️ Exempt it** links are built by appending the proposed entry
with `yq` rather than by string concatenation, so the prefilled result is valid
YAML whatever shape the file is in — including `exemptions: []`, where a
textual append would produce a broken document. Header comments are preserved.

Because GitHub's `value=` parameter replaces the *whole* file, each link embeds
the entire exemption file. At ~22 entries that is ~2.8 KB per URL against a
practical ~8 KB ceiling, leaving room for roughly 60 entries. Past that the
links will start to break and should be swapped for a plain edit link without
prefill.

## Setup

1. Set `ORG_NAME` and `POLICY_URL` in the workflow's `env:` block.
2. Add `ORG_MONITOR_TOKEN` as a repository secret. **Read access only** — the
   workflow never writes to any repository, and no write scope is required.
3. Commit `.github/security-md-exemptions.yaml` alongside the workflow. It is
   read via `actions/checkout`, so it must live in the same repository; a
   missing file fails the audit step.
4. Trigger once manually and review the summary before letting the nightly
   schedule take over.

## Behavior notes

- **The run fails when any in-scope repo is missing `SECURITY.md`.** Expect the
  first run to be red. To use the workflow as a dashboard instead of a gate,
  change the final `exit 1` in the report job to a warning.
- **Repos with zero commits** have no default branch. They cannot hold a
  `SECURITY.md` and cannot receive a pull request, so they are reported as
  `⚠️ empty repo, no commits` rather than given a dead link. Consider exempting
  them outright.
- **Archived and private repos are out of scope** by construction, filtered in
  the GraphQL query rather than after the fact.
- **Presence, not content.** The audit checks that `SECURITY.md` exists; it does
  not verify the file actually links back to the canonical policy. A repo with a
  divergent hand-written copy counts as compliant.
- **New repos are the main source of drift.** Compliance correlates strongly
  with repo age — recently created repos are the least likely to have the file.
  A nightly audit will keep finding new gaps; seeding the stub at repo-creation
  time is the durable fix, with this workflow as the safety net.
