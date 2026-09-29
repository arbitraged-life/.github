# Reusable auto-merge gate contract

`auto-merge-reusable.yml` calls `merge_gate.py` for one bounded scan. The Actions-only path is the default and requires exactly one successful named Actions workflow run for the PR's current head SHA. Advisory Actions workflows are reported as optional observations; they never substitute for or block a required workflow. The gate retains the configured opt-in label and merge method, requires GitHub to report `MERGEABLE`, re-reads the PR immediately before merge, and uses `gh pr merge --match-head-commit <sha>` as a server-side expected-head guard.

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

When provider mode is explicitly enabled, it replaces the Actions mandatory gate with the complete provider-required workflow set. Each item must have exactly one check run named `CircleCI / <workflow>` on the repository's current PR head SHA, with the configured GitHub App ID, `status=completed`, `conclusion=success`, and an `external_id` matching `<pipeline UUIDv4>:<workflow UUIDv4>:attempt-<positive integer>:pr-<current PR number>:<correlation UUIDv4>`. The gate requires canonical lowercase UUIDv4 syntax for the Circle pipeline ID, workflow ID, and correlation UUID, a positive attempt, and a receipt PR number equal to the current PR. Each workflow is independently routed, so its pipeline ID, attempt, and correlation UUID may differ from other required workflows. The check-run query is scoped to the target repository and current SHA; if GitHub's `total_count` exceeds the returned page, provider evidence is rejected rather than evaluated from a truncated page. The bridge validates repository/workflow/event/PR/SHA/attempt against its per-workflow D1 receipt, and the check's `head_sha` is compared against the live PR head. Missing, duplicate/ambiguous, stale, pending, failed, skipped, cancelled, untrusted, or malformed evidence fails closed. Arbitrary commit statuses are not read or trusted. The provider setting remains disabled by default.

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

An external scheduler can invoke one pass on a bounded cadence without consuming hosted Actions minutes. The pass considers at most 100 labeled open PRs and allows no more than 20 entries in either workflow list; each GitHub CLI/API request times out after 30 seconds. The reusable Actions job has a 15-minute ceiling. The pass performs no waiting/polling loops and exits nonzero on unavailable required GitHub API or malformed configuration. Advisory API failures are logged but do not block required gates. Callers should schedule periodic recovery and trigger passes on label, head-update/reopen, and required-check completion events. Multiple callbacks are safe to retry: only open labeled PRs are considered, all evidence is tied to the current SHA, and merge requests carry the expected SHA; after a successful merge the PR is no longer open. The coordinator needs `contents:write`, `pull-requests:write`, `checks:read`, and `actions:read` permissions. Build/test jobs must not receive these write permissions or invoke this coordinator with their credentials.

## Rollback

For rollback, remove provider inputs or leave `provider-enabled: false`, then call the reusable workflow with the existing `gate-workflow`, label, and merge method. This restores the Actions-only required gate; no provider check or status needs to be deleted. Do not change repository protection or merge unrelated PRs as a rollback test.

## Local behavioral tests and smoke

Run the targeted standard-library suite after integration:

```sh
python3 -m unittest discover -s .github/scripts -p 'test_merge_gate.py' -v
```

A no-write coordinator smoke can use a read-only token against an empty/test repository and confirm it completes a bounded scan without merging. A real disposable opt-in PR/cloud run is still required before declaring consumer acceptance; it was not run here, and provider mode remains disabled.
