---
model: haiku
---
Merge a PR/MR (GitHub, GitLab or Azure DevOps) after review findings have already been documented and fixed.

Assumes the workflow: `/pr` (document) → `/fix` (fix findings) → `/merge` (this).

## Usage

```
/merge              Merge current branch's PR
/merge #123         Merge specific PR by number
/merge --dry-run    Show what would happen without merging
```

## Forge & Tracker

Detect the forge, `$BASE` and the tracker exactly as in `/pr` ([Forge & Tracker Detection](pr.md#forge--tracker-detection)). Use `$BASE` wherever this command says `main`. On GitLab and Azure DevOps, substitute:

| Step | GitLab | Azure DevOps |
|------|--------|--------------|
| Identify | `glab mr view [N] -F json` | `az repos pr show --id N -o json` (current branch: `az repos pr list --detect true --source-branch "$BRANCH" --status active`) |
| Pre-flight | `detailed_merge_status` must be `mergeable`; also `draft`, `sha` | `isDraft` false, `mergeStatus` `succeeded`; `az repos pr reviewer list --id N` — another reviewer has `vote` ≥ 5; `az repos pr policy list --id N` — every blocking policy `approved` |
| Not approved yet | Stop; offer `glab mr merge N --auto-merge --yes` | Stop; offer `az repos pr update --id N --auto-complete true --delete-source-branch true` |
| Merge | `glab mr merge N --sha <sha> --remove-source-branch --yes` | `az repos pr update --id N --status completed --delete-source-branch true` |

- GitLab `detailed_merge_status` values like `not_approved`, `ci_still_running`, `discussions_not_resolved` or `draft_status` explain why it isn't mergeable — report the value as-is.
- `--sha` pins the merge to the reviewed head commit; if someone pushed since, the merge fails — report it, don't retry.
- Azure DevOps votes: `10` approved, `5` approved with suggestions, `0` no vote, `-5` waiting for author, `-10` rejected. Your own vote never counts.
- **Skip step 3 (rebase + release commit) on GitLab and Azure DevOps.** Pushing after approval resets approvals, and the client owns its release process. The forge merges exactly what was reviewed.

## Process

### 1. Identify PR

```bash
# Current branch's PR
gh pr view --json number,title,headRefName,baseRefName

# Or specific PR
gh pr view 123 --json number,title,headRefName,baseRefName
```

### 2. Pre-flight Checks

```bash
gh pr view $PR --json mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,isDraft
```

| Check | Must Be |
|-------|---------|
| `isDraft` | `false` |
| `mergeable` | `MERGEABLE` |
| `mergeStateStatus` | `CLEAN` or `UNSTABLE` |
| `reviewDecision` | `APPROVED` (if required) |
| `statusCheckRollup` | All `SUCCESS` or `NEUTRAL` |

If `mergeStateStatus` is `UNSTABLE` (checks still running), wait for CI before proceeding:

```bash
# Wait strategy: sleep 3 min upfront (CI typically takes 3-5 min), then poll every 30s.
# Do NOT poll in a tight loop from the start — it adds noise and wastes time.
sleep 180
for i in {1..10}; do
  STATUS=$(gh pr view $PR --json statusCheckRollup -q \
    '[.statusCheckRollup[] | select(.status != "COMPLETED")] | length')
  [[ "$STATUS" == "0" ]] && break
  echo "Still waiting... ($i/10)"
  sleep 30
done
gh pr view $PR --json statusCheckRollup -q \
  '.statusCheckRollup[] | "\(.name): \(.status) (\(.conclusion))"'
```

Stop and report if any check fails — do not proceed.

**Output when ready:**

```
Pre-flight Check Passed
═══════════════════════════════════════════════════════════════

PR #123: feat(auth): add OAuth support

✓ Not a draft
✓ No merge conflicts
✓ Approved
✓ CI passing (5/5)

Linked issues that will close: #101, #98
Branch to delete: feat/oauth-support

Proceed? [Y/n]
```

### 3. Rebase on Main, Then Update Changelog & Version

**Always rebase before writing the version bump.** Other PRs may have merged since
you started — if you bump to X.85 before rebasing and main already has X.85, you get a
conflict and a wasted CI run.

```bash
# 1. Discard any local Cargo.lock drift before rebase
git restore Cargo.lock 2>/dev/null || true

# 2. Rebase on main
git fetch origin
git rebase origin/main

# 3. Read the CURRENT version from main, then increment
# (do not assume you know the version — another PR may have bumped it)
CURRENT=$(grep '^version' Cargo.toml | head -1 | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')
echo "Current version on main: $CURRENT"
```

Determine version bump from branch type:
- `feat/*` → minor bump (0.X.0)
- `fix/*` → patch bump (0.0.X)
- All others → skip version bump entirely (no release commit)

Update `CHANGELOG.md` and bump version in the appropriate locations for the project type:

**Python:**
1. `src/<package>/__init__.py` — `__version__ = "X.Y.Z"`
2. `pyproject.toml` — `version = "X.Y.Z"`
3. `README.md` — version badge or `## Version: X.Y.Z`
4. `CHANGELOG.md` — new `## [X.Y.Z] - YYYY-MM-DD` section

**Rust:**
1. `Cargo.toml` — `version = "X.Y.Z"` (workspace root if applicable)
2. `README.md` — version badge or `## Version: X.Y.Z`
3. `CHANGELOG.md` — new `## [X.Y.Z] - YYYY-MM-DD` section

If no version files exist, only update `CHANGELOG.md`.

```bash
# Python
git add CHANGELOG.md README.md src/*/ pyproject.toml

# Rust
git add CHANGELOG.md README.md Cargo.toml Cargo.lock

# Use --no-verify: release commit only touches CHANGELOG/version files, not code.
# Running the full test suite here is wasteful and adds ~60s per commit.
git commit --no-verify -m "chore: release vX.Y.Z"
git push
```

### 4. Detect Linked Issues

```bash
gh pr view $PR --json body,commits --jq '
  [.body, (.commits[].messageBody // "")] |
  join(" ") |
  match("(closes?|fixes?|resolves?)\\s*#([0-9]+)"; "gi") |
  .captures[1].string
' | sort -u
```

Warn if no linked issues found.

With `tracker: jira`, collect `<KEY>-N` references from the PR title, body and branch name instead.

### 5. Merge the PR

On GitHub, always use the GitHub API — `gh pr merge` fails with worktrees (GitLab / Azure DevOps: use the Merge command from the table):

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)

gh api -X PUT repos/$REPO/pulls/$PR/merge \
  -f merge_method=merge

# Delete remote branch
BRANCH=$(gh pr view $PR --json headRefName -q .headRefName)
gh api -X DELETE repos/$REPO/git/refs/heads/$BRANCH 2>/dev/null || true
```

### 6. Sync Local Repository

```bash
BRANCH=$(gh pr view $PR --json headRefName -q .headRefName)

# Find worktree with this branch checked out
WORKTREE=$(git worktree list --porcelain | grep -B2 "branch refs/heads/$BRANCH" | grep "^worktree" | awk '{print $2}')

git fetch --prune

# Pull main in whichever worktree has it
BASE_WORKTREE=$(git worktree list --porcelain | grep -B2 "branch refs/heads/main" | grep "^worktree" | awk '{print $2}')
if [[ -n "$BASE_WORKTREE" ]]; then
  git -C "$BASE_WORKTREE" pull origin main
fi

# Delete local feature branch
if [[ -n "$WORKTREE" ]]; then
  AGENT_BRANCH=$(git -C "$WORKTREE" branch --show-current | grep agent || echo "main")
  git -C "$WORKTREE" checkout "$AGENT_BRANCH" 2>/dev/null && \
    git branch -D "$BRANCH" 2>/dev/null || true
else
  git branch -D "$BRANCH" 2>/dev/null || true
fi
```

### 7. Verify Linked Issues

```bash
for ISSUE in $LINKED_ISSUES; do
  gh issue view $ISSUE --json number,state,stateReason -q \
    '"Issue #\(.number): \(.state) (\(.stateReason // "N/A"))"'
done
```

If an issue is still open, offer to close it manually:

```bash
gh issue close $ISSUE --reason completed --comment "Closed via PR #$PR"
```

**Jira:** do not transition or close anything. The issue stays in the `review` status (e.g. `Acceptatie`) until the user accepts it. End with a reminder:

```
⏰ <KEY>-123 is in Acceptatie — move it to Done once accepted:
   acli jira workitem transition --key <KEY>-123 --status "Done"
```

## Output

```
PR Merged Successfully
═══════════════════════════════════════════════════════════════

✓ CHANGELOG.md updated — [0.4.0] section added
✓ Version bumped: 0.3.1 → 0.4.0
✓ Release commit: abc1233 chore: release v0.4.0
✓ PR #123 merged into main
✓ Remote branch 'feat/oauth-support' deleted
✓ Local branch 'feat/oauth-support' deleted
✓ main pulled and up to date
✓ Stale refs pruned

Linked Issues:
  ✓ #101 closed (completed)
  ✓ #98 closed (completed)

Repository is clean and up to date.
```

## Dry Run

```
/merge --dry-run

Dry Run: PR #123
═══════════════════════════════════════════════════════════════

Would perform:
  1. Update CHANGELOG.md + bump version to 0.4.0
  2. Commit: chore: release v0.4.0
  3. Merge PR #123 into main
  4. Delete remote branch: origin/feat/oauth-support
  5. Delete local branch: feat/oauth-support
  6. Pull main, prune stale refs

Issues that would close: #101, #98

No changes made. Run without --dry-run to execute.
```

## Edge Cases

**Local branch has unpushed commits:** warn and ask before deleting.

**PR from fork:** skip remote branch deletion (no permission).

**Protected branch / squash not allowed:** fall back to merge commit.

## Safety

- Never merges draft PRs
- Never merges with failing required checks
- Never approves PRs, never uses `--bypass-policy`, never merges without another reviewer's approval where required
- Never merges with unresolved conflicts
- Never force-deletes unmerged local branches
- Always confirms before proceeding
