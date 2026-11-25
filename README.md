# renovate-config

The shared [Renovate](https://docs.renovatebot.com/) preset for JanWelker's
repositories. A repository opts in with:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>JanWelker/renovate-config"]
}
```

## Policy

| Rule | Setting | Why |
| --- | --- | --- |
| Release age | `minimumReleaseAge: 3 days` | Long enough for an upstream to pull a broken or compromised release |
| PR creation | `prCreation: not-pending` | The PR opens only once `renovate/stability-days` passed; GitHub's auto-merge only watches the required checks, so a digest PR opened earlier merged before its age |
| Renovate's own checks | `internalChecksAsSuccess: true` | CI runs only on the PR, so a passed `renovate/stability-days` is the branch's only status; without this Renovate calls it pending and holds the PR another 25 hours (`prNotPendingHours`) |
| Release age without timestamps | No release age, `prCreation: immediate` and `automerge: false` for third-party images on `ghcr.io`, `gcr.io`, `quay.io`, ECR Public and `registry.k8s.io` | No release date, so no age can be met; a human stands in for it, skipping it avoids `timestamp-optional` warnings, and with no status `not-pending` would hold the PR indefinitely |
| Trusted publishers' digests | `automerge: true` for digest updates from the publishers listed in `default.json` | A digest is a rebuild of a tag already accepted; waiting for a human left base-image security rebuilds open for weeks. New tags from them still wait |
| Own images | No release age and `prCreation: immediate` for `ghcr.io/janwelker/**` and `ghcr.io/somuch-cloud/**` | Nothing to wait out on our own releases; without a `renovate/stability-days` status the branch has none, so `not-pending` held the PR 25 hours and every new release restarted the wait |
| Patch, minor, digest | `automerge: true` | Merges once the required checks are green |
| Major, and a minor below 1.0 | `automerge: false`, review requested; a major opens at 19:00 Europe/Zurich | A major is where a migration hides, and below 1.0 the minor is the breaking release; merging it is the approval |
| Lock file maintenance | Mondays, `prCreation: immediate`, `automerge: true` | It moves every transitive package at once with no timestamp to age; the required checks are the gate, as for a patch |
| Security fixes | Renovate's `vulnerabilityAlerts` defaults | Ignore the schedule, the release age and the PR limits, so they open at any time |
| npm for lock files | `constraints: { npm: "<12" }` | npm 12's `allow-remote` default refuses registry tarballs while regenerating `package-lock.json` (`EALLOWREMOTE`); lift once npm fixes it ([npm/cli#9347](https://github.com/npm/cli/issues/9347)) |
| TypeScript | `allowedVersions: <7` | 7 is the native port; the toolchains here are not ready for it |
| One PR per concern | Renovate's default branching, `separateMultipleMajor` | Group only what must move together, in the repository's own config |

## Required checks

A repository's ruleset on the default branch requires every check that runs
on its pull requests, by name, from the GitHub Actions app (`integration_id`
15368; CodeQL's `CodeQL` result comes from 57789). Without a required check,
GitHub's auto-merge merges a red PR.

| Rule | Why |
| --- | --- |
| A check counts when it runs on every pull request | One that does not start stays "Expected" and blocks the PR forever |
| Path filters are job conditions, not `on.pull_request.paths` | A job skipped by its `if:` reports `skipped`, which passes; a workflow filtered out never reports |
| A new workflow or job is added to the ruleset in the same change | Otherwise it runs, fails, and blocks nothing |

## What a repository needs besides this preset

- A ruleset on the default branch requiring its checks, as in
  [Required checks](#required-checks).
- A repository with more than one writer also requires one approval
  (`required_approving_review_count: 1`) in that ruleset. The checks come from
  the pull request's own workflow files, so they guard against broken
  updates, not against a PR author, who can rewrite or delete them.
- Dependabot alerts on, Dependabot security updates off: Renovate reads the
  alerts and opens the fix PRs itself.
- A private repository on GitHub Free has no rulesets, so it sets
  `"platformAutomerge": false`; Renovate then merges only after every check
  passed.
- A repository without CI on pull requests sets `"automerge": false`.
