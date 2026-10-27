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
| PR creation | `prCreation: not-pending` | The PR opens only once `renovate/stability-days` passed; GitHub's auto-merge only watches `ci-ok`, so a digest PR opened earlier merged before its age |
| Release age without timestamps | No release age and `automerge: false` for third-party images on `ghcr.io`, `gcr.io`, ECR Public and `registry.k8s.io` | These registries report no release date, so the age could never be met and the update would wait forever; a human stands in for the age, and skipping it instead of `timestamp-optional` keeps the dashboard free of its warnings |
| Own images | No release age for `ghcr.io/janwelker/**` | Nothing to wait out on our own releases |
| Patch, minor, digest | `automerge: true` | Merges once the required `ci-ok` check is green |
| Major, and a minor below 1.0 | `automerge: false`, review requested; a major opens at 19:00 Europe/Zurich | A major is where a migration hides, and below 1.0 the minor is the breaking release; merging it is the approval |
| Lock file maintenance | Mondays, `automerge: false`, review requested | It moves every transitive package at once with no timestamp to age |
| Security fixes | Renovate's `vulnerabilityAlerts` defaults | Ignore the schedule, the release age and the PR limits, so they open at any time |
| npm for lock files | `constraints: { npm: "<12" }` | npm 12's `allow-remote` default refuses registry tarballs while regenerating `package-lock.json` (`EALLOWREMOTE`); lift once npm fixes it ([npm/cli#9347](https://github.com/npm/cli/issues/9347)) |
| TypeScript | `allowedVersions: <7` | 7 is the native port; the toolchains here are not ready for it |
| One PR per concern | Renovate's default branching, `separateMultipleMajor` | Group only what must move together, in the repository's own config |

## The ci-ok check

`wait-for-checks.yaml` watches every check run and commit status on the head
commit, Renovate's `renovate/stability-days` and `renovate/artifacts`
included.

| Outcome | When |
| --- | --- |
| Pass | After `settle-seconds` (120), nothing pending, nothing new since the last poll, every `required` name succeeded |
| Fail at once | Any check or status ends anything but `success`, `skipped` or `neutral` (CodeQL reports neutral); a job cancelled by a newer one in its concurrency group is skipped |
| Fail at `deadline-minutes` (80) | Something still pending, a `required` name missing, or no other check reported at all |

| Input | Default | Use |
| --- | --- | --- |
| `runs-on` | `ubuntu-latest` | Runner label |
| `required` | empty | Check names or status contexts, one per line, that must appear and succeed; catches a workflow that registers late or was deleted |
| `settle-seconds` | `120` | Shortest watch before passing |
| `deadline-minutes` | `80` | Longest wait; the job times out at 90 |

A red `ci-ok` on Renovate PRs across repositories after a new tag is the sign
the check got too strict; its log names the check it waited for. Re-run
`ci-ok` after re-running a failed check.

## What a repository needs besides this preset

- A `ci-ok` job calling `wait-for-checks.yaml` from this repository, pinned to a
  tag, which runs on every pull request, required by a ruleset on the default
  branch. Without a required check, GitHub's auto-merge merges a red PR. Copy
  [`ci-ok.yaml`](.github/workflows/ci-ok.yaml), including its
  `statuses: read`. A repository on self-hosted runners passes their label as
  `runs-on`; the wait needs only `curl` and `jq`.
- A repository with more than one writer also requires one approval
  (`required_approving_review_count: 1`) in that ruleset. `ci-ok` comes from
  the pull request's own workflow files, so it guards against broken
  updates, not against a PR author, who can rewrite or delete it.
- Dependabot alerts on, Dependabot security updates off: Renovate reads the
  alerts and opens the fix PRs itself.
- A private repository on GitHub Free has no rulesets, so it sets
  `"platformAutomerge": false`; Renovate then merges only after every check
  passed.
- A repository without CI on pull requests sets `"automerge": false`.
