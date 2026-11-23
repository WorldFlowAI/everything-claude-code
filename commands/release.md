---
description: Cut a release. Invokes the release-manager agent to compute the semver bump from changes since the last tag, update versions and CHANGELOG, verify, commit and tag locally, then publish (push tag, GitHub release, optional registry publish) only after your approval.
argument-hint: "[patch|minor|major|<x.y.z>] [--pre <rc|beta|alpha>] [--dry-run]"
---

# Release Command

Invokes the **release-manager** agent.

## Usage

```
/release              # infer the bump from commits/PRs since the last tag
/release minor        # force a bump level
/release 2.0.0        # explicit version
/release --pre rc     # pre-release, e.g. 1.4.0-rc.1
/release --dry-run    # show the version, changelog, and checks; change nothing
```

## What Happens

1. **Preflight:** clean tree, on the default branch, synced with the remote, last tag found
2. **Changes collected:** commits and merged PRs since the last tag, classified as breaking / added / fixed / security / internal
3. **Version decided:** SemVer from the highest change (or `$ARGUMENTS`), checked against the actual API diff (`cargo semver-checks` for Rust libraries)
4. **Files updated:** version sources, lockfiles, `CHANGELOG.md`
5. **Verified:** full CI-equivalent checks + publish dry-run + release build
6. **Committed and tagged locally:** `chore(release): vX.Y.Z` + annotated tag
7. **Approval gate:** you review the summary, then approve pushing the tag, creating the GitHub release, and (separately) registry publishing

Nothing leaves your machine before step 7. If you decline, the summary includes the undo command.

## Arguments

`$ARGUMENTS`:
- `patch` | `minor` | `major`: force the bump level
- `<x.y.z>`: explicit version (must be greater than the last release)
- `--pre <label>`: pre-release suffix
- `--dry-run`: stop after step 5 and revert any file changes

## Related

- `/ship`: delivers individual features as PRs; `/release` packages merged work into a version
- `devops` agent: if tags trigger deployments, it owns the pipeline and rollback steps
