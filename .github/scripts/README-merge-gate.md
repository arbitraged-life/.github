# Reusable auto-merge gate contract

`auto-merge-reusable.yml` calls `merge_gate.py` for one bounded scan. The Actions-only path is the default and requires exactly one successful named Actions workflow run for the PR's current head SHA. Advisory Actions workflows are reported as optional observations; they never substitute for or block a required workflow. The gate retains the configured opt-in label and merge method, requires GitHub to report `MERGEABLE`, re-reads the PR immediately before merge, and uses `gh pr merge --match-head-commit <sha>` as a server-side expected-head guard.

The reusable job checks out its coordinator from `job.workflow_sha`, the exact commit defining the called workflow, rather than mutable `main`. Consumers must pin their `uses:` reference to an immutable commit for stable behavior. GitHub documents these `job.workflow_*` fields for reusable workflows on GitHub.com; they are unavailable on GitHub Enterprise Server. The label re-read is best-effort: GitHub's merge endpoint only provides an atomic head-SHA guard, not a label precondition. Keep the opt-in gate disabled unless that residual race is acceptable or enforcement is provided by independently verified branch protection.

## Inputs and decisions

The reusable workflow inputs are:

- `gate-workflow`: required Actions workflow filename; the only mandatory gate in default mode.
- `label`: opt-in PR label (default `auto-merge`). It is checked again immediately before merge.
- `merge-method`: `merge`, `squash`, or `rebase` (default `squash`).
- `advisory-workflows`: JSON array of Actions workflow filenames, e.g. `["security.yml"]`; these remain advisory.
- `provider-enabled`: explicit boolean opt-in, default `false`.
- `trusted-publisher-app-id`: exact GitHub App ID expected on every required provider check. This value must come only from separately reviewed trusted consumer configuration.
- `provider-required-workflows`: JSON array of exact Circle result names without the prefix, e.g. `["ci","integration"]` for `CircleCI / ci` and `CircleCI / integration`.
- `max-prs`: maximum labeled PRs visited by a single pass (1–100; default 100).

Provider mode explicitly replaces the Actions mandatory gate with the complete configured provider-required workflow set; the coordinator must not infer that set from available green checks. For each required `CircleCI / <workflow>` name, it requires exactly one completed successful check on the current PR head from the configured GitHub App ID. Its `external_id` must match `<pipeline UUIDv4>:<workflow UUIDv4>:attempt-<positive integer>:pr-<current PR number>:<correlation UUIDv4>`. UUIDs must be canonical lowercase UUIDv4; independently routed workflows may use different pipeline IDs, attempts, and correlations. The bridge validates repository/workflow/event/PR/SHA/attempt against its per-workflow D1 receipt. Missing, duplicate, stale, pending, failed, skipped, cancelled, untrusted, or malformed evidence fails closed; arbitrary commit statuses are not consulted.

Provider check runs are paged with `filter=all` (not only the newest check by each publisher), up to 10 pages/1,000 runs. A changing result count, incomplete page, or more than 1,000 checks aborts the pass without merging on incomplete evidence. Once all pages are collected, duplicate required check names are ambiguous and rejected. A page may still change status after it was read; independent required-check protection is necessary for a live merge, especially on private repositories where protection APIs may be unavailable.

### Provider acceptance blocker

Do not set `provider-enabled: true` today. The bridge exposes the attempt, PR number, and correlation UUID in `external_id` and confirms its D1 receipt checks repository/workflow/event/PR/SHA/attempt. However, no trusted GitHub App ID is provisioned or verified, and deployment remains disabled until its App installation/ID and token are provisioned. A trusted publisher identity, a live verified result run, and separate authorization are still required before enabling provider mode. No consumer integration or live-provider verification is claimed here.

## Coordinator invocation

The same script is the bounded coordinator interface for a non-Actions runner with Python 3, GitHub CLI (`gh`), and an authorized `GH_TOKEN` in its environment:

```sh
python3 .github/scripts/merge_gate.py \
  --repo owner/repository \
  --gate-workflow ci.yml \
  --label auto-merge \
  --merge-method squash \
  --max-prs 100
```

An external scheduler can invoke one pass on a bounded cadence without consuming hosted Actions minutes. The pass considers at most 100 labeled open PRs, up to 1,000 check runs per head, and no more than 20 entries in either workflow list; each GitHub CLI/API request times out after 30 seconds. The reusable Actions job has a 15-minute ceiling. The pass performs no waiting/polling loops and exits nonzero on unavailable required GitHub API or malformed configuration. Advisory API failures are logged but do not block required gates. Callers should schedule periodic recovery and trigger passes on label, head-update/reopen, and required-check completion events. Multiple callbacks are safe to retry: only open labeled PRs are considered, all evidence is tied to the current SHA, and merge requests carry an expected-head guard. This script is the coordinator implementation, **not a deployed external schedule**; rollout still requires a trusted scheduler/executor, scoped merge credential, required-check protection, and live disposable-PR evidence.

## Rollback

For rollback, remove provider inputs or leave `provider-enabled: false`, then call the reusable workflow with the existing `gate-workflow`, label, and merge method. This restores the Actions-only required gate; no provider check or status needs to be deleted. Do not change repository protection or merge unrelated PRs as a rollback test.

## Local behavioral tests and smoke

Run the targeted standard-library suite after integration:

```sh
python3 -m unittest discover -s .github/scripts -p 'test_merge_gate.py' -v
```

A no-write coordinator smoke can use a read-only token against an empty/test repository and confirm it completes a bounded scan without merging. A real disposable opt-in PR/cloud run is still required before declaring consumer acceptance; it was not run here, and provider mode remains disabled.
