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
| Patch, minor, digest, lock file | `automerge: true` | Merges once the required `ci-ok` check is green |
| Major | `automerge: false`, opened at 19:00 Europe/Zurich, review requested | A major is where a migration hides; merging it is the approval |
| Security fixes | Renovate's `vulnerabilityAlerts` defaults | Ignore the schedule, the release age and the PR limits, so they open at any time |
| TypeScript | `allowedVersions: <7` | 7 is the native port; the toolchains here are not ready for it |
| One PR per concern | Renovate's default branching, `separateMultipleMajor` | Group only what must move together, in the repository's own config |

## What a repository needs besides this preset

- A `ci-ok` job calling `wait-for-checks.yaml` from this repository, pinned to a
  tag, which runs on every pull request and fails when any other check
  on the head commit fails, required by a ruleset on the default branch.
  Without a required check, GitHub's auto-merge merges a red PR.
- Dependabot alerts on, Dependabot security updates off: Renovate reads the
  alerts and opens the fix PRs itself.
- A private repository on GitHub Free has no rulesets, so it sets
  `"platformAutomerge": false`; Renovate then merges only after every check
  passed.
- A repository without CI on pull requests sets `"automerge": false`.
