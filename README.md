# Shared repository automation

`ci/homelab-pull-request.yml` and `ci/agent-framework-pull-request.yml` are
CircleCI GitHub App configurations for private-repository pull requests. Their
configuration source is this repository at `refs/heads/main`; the event and
checkout sources are the corresponding private repository. Do not point a PR
trigger at a config stored in the PR repository: it could execute PR-authored
CI policy. The GitHub App trigger must supply `config_ref: main` and must not
override the checkout ref when event and checkout source are the same repo.

Each workflow validates the event, checkout SHA, and live PR ref before and
after its checks. Before running PR code, its preflight refuses changes to
security verifier scripts, their policy files, or shipped skillspector
baselines relative to the PR's trusted base. Land intentional security-policy
changes separately, review them on `main`, then rebase the feature PR.
Homelab runs policy auditors before installing Node packages without lifecycle
scripts; its router security regressions come from the trusted base and run
against the PR implementation before PR-authored tests. Agent-framework runs
its protected purity checker before PR-authored pytest modules can modify the
working tree. The framework security image uses Python 3.12 because pinned
Skillspector v2.9.6 requires Python 3.12 or newer. Gitleaks loads its rules
from the trusted base commit; preflight
rejects changes to the root `.gitleaksignore` because Gitleaks also reads that
file from the scan source. Inline `gitleaks:allow` directives are disabled, and
the scanner covers the PR commit range.

Framework Node jobs use `cimg/node:24.15.0` to match the Node 24 LTS contract
in agent-framework's package and Actions configuration. The Python audit
environment includes pinned PyYAML for the skill metadata validator.

No write token, self-hosted Actions runner, merge controller, or privileged
remediation belongs in these untrusted-PR jobs. Confirm the CircleCI Checks
GitHub App owns a real check run on the current PR head and that checkout used
that exact SHA before replacing any existing required check or enabling a
privileged follow-up. Private GitHub Free repositories cannot enforce branch
protection; a passing check is evidence, not a merge permission boundary.
