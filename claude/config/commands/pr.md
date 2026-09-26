---
model: haiku
---
Create and document a pull request. In the workflow `/review → /pr → /fix → /merge`,
this step creates the PR and records review findings in the body so `/fix` has a clear target.

## Usage

```
/pr                  Create PR from current branch to the default branch
/pr draft            Create as draft PR
/pr <base>           Create PR targeting specific base branch
/pr list             List open PRs with status
/pr #123             Show details for specific PR
```

Works on GitHub, GitLab (merge requests) and Azure DevOps — see [Forge & Tracker Detection](#forge--tracker-detection).

---

## Forge & Tracker Detection

**Forge** — from `git remote get-url origin`:

| Remote contains | Forge | CLI |
|-----------------|-------|-----|
| `github.com` | GitHub | `gh` |
| `dev.azure.com`, `ssh.dev.azure.com`, `visualstudio.com` | Azure DevOps | `az repos` |
| anything else | GitLab (self-hosted) | `glab` |

**Base branch** — default to the repo's default branch, not a hard-coded `main`:

```bash
BASE=$(git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||')
BASE=${BASE:-main}
```

Use `$BASE` wherever this command says `main` / `origin/main`. `/pr <base>` overrides it.

**Tracker** — from the `## Workflow` block in `CLAUDE.local.md` (see `/take`). With `tracker: jira`, issue references are Jira keys (`<KEY>-123`) and the `review` transition is applied after creation (step 6).

**Command mapping** (GitHub commands in the steps below; substitute for other forges):

| Step | GitHub | GitLab | Azure DevOps |
|------|--------|--------|--------------|
| Existing PR? | `gh pr view` | `glab mr list --source-branch "$BRANCH"` | `az repos pr list --detect true --source-branch "$BRANCH" --status active` |
| Create | `gh pr create --title … --body … --base "$BASE"` | `glab mr create --title … --description-file - --target-branch "$BASE" --yes` (body on stdin) | `az repos pr create --detect true --title … --description "$BODY" --target-branch "$BASE"` |
| Draft | `--draft` | `--draft` | `--draft true` |
| List | `gh pr list …` | `glab mr list -F json` | `az repos pr list --detect true --status active -o json` |
| View | `gh pr view 123 …` | `glab mr view 123 --comments -F json` | `az repos pr show --id 123 -o json` |

Forge notes:
- GitLab calls them **merge requests** (`!123`); use that wording in output.
- Azure DevOps PR descriptions are capped at **4000 characters** — trim the Changes section to fit, never drop Review Findings.
- Azure DevOps PRs require approval from another reviewer; `/pr` only creates, it never approves or completes.

---

## Create Mode (default)

### 0. Pre-flight: Validate Branch

Never create a PR from `agent-XX` base branches (worktree bases) or from `main` with no new commits.

```bash
BRANCH=$(git branch --show-current)

if [[ "$BRANCH" =~ ^agent-[0-9]+$ ]]; then
  echo "✗ Cannot create PR from agent-XX base branch."
  echo "  Create a feature branch first: git checkout -b feat/your-feature"
  exit 1
fi

if [[ "$BRANCH" == "main" ]]; then
  COMMITS_AHEAD=$(git rev-list --count origin/main..HEAD)
  if [[ "$COMMITS_AHEAD" -eq 0 ]]; then
    echo "✗ No commits ahead of main. Create a feature branch first."
    exit 1
  fi
fi
```

Also warn if:
- Uncommitted changes exist (`git status --short`)
- Branch not pushed to remote (`git log origin/$(git branch --show-current)..HEAD`)
- PR already exists for this branch (`gh pr view 2>/dev/null`, or the forge's "Existing PR?" command)

### 1. Gather Context

```bash
git branch --show-current
git log origin/$BASE..HEAD --oneline
git diff origin/$BASE...HEAD --stat
git diff origin/$BASE...HEAD
```

### 2. Generate Title

Follow conventional commit format: `type(scope): concise description`

Derive from branch name:
- `feat/add-dark-mode` → `feat(ui): add dark mode support`
- `fix/123-login-bug` → `fix(auth): resolve login redirect issue`

### 3. Generate PR Body

Include findings from any prior `/review` run in the current session as actionable items.

```markdown
## Summary

[2-3 bullet points: what and why]

## Changes

### Added
- ...

### Changed
- ...

### Fixed
- ...

## Review Findings

[If `/review` was run: list findings by severity for `/fix` to address.
 If no prior review: omit this section.]

| Severity | File | Issue |
|----------|------|-------|
| warning  | src/foo.py:42 | Missing error handling |
| suggestion | tests/ | Add edge case for X |

## Testing

- [ ] Unit tests added/updated
- [ ] Manual testing performed
- [ ] Edge cases considered

## Related Issues

Closes #N

---
🤖 *Generated with [Claude Code](https://claude.ai/code)*
```

### 4. Detect Related Issues

Check branch name, commit messages, and TODO comments for `#N` references.
Use `Closes #N` for auto-close on merge, `Related to #N` otherwise.

With `tracker: jira`, look for `<KEY>-N` instead (e.g. branch `feat/<KEY>-123-desc`). Put the key in the title — `feat(api): add export (<KEY>-123)` — and write `Related: <KEY>-123` under Related Issues. Jira issues are not auto-closed.

### 5. Create PR

```bash
gh pr create \
  --title "type(scope): description" \
  --body "$(cat <<'EOF'
...
EOF
)"
```

Add `--draft` for work-in-progress. Add `--base <branch>` for non-main targets.
On GitLab or Azure DevOps, use that forge's Create command from the mapping table.

### 6. Move Jira Issue to Review (Jira only)

Skip for draft PRs. Otherwise, after the PR/MR exists, ask the user, then apply the `review` transition from the `## Workflow` block:

```bash
acli jira workitem transition --key <KEY>-123 --status "<review>" --yes
```

If the transition fails (workflow rule, wrong status name), report it — the PR stays as is; do not retry or pick another status.

**Output:**

```
PR Created
═══════════════════════════════════════════════════════════════

PR #124: feat(auth): add OAuth support
URL: https://github.com/owner/repo/pulls/124

Review findings documented: 2 warnings, 3 suggestions
Linked issues: Closes #101, #98
Jira: <KEY>-123 → <review>          (Jira only)

Next: /fix — address the review findings
```

---

## List Mode (`/pr list`)

```bash
gh pr list --state open --json number,title,author,createdAt,reviewDecision,isDraft,labels
```

**Output:**

```
Open Pull Requests
═══════════════════════════════════════════════════════════════

#124 feat(auth): add OAuth support                @me
     Created: just now | Draft | 2 warnings to fix

#138 fix(api): handle timeout errors              @bob
     Created: 1 week ago | Changes requested

Summary: 2 open PRs
```

---

## View Mode (`/pr #123`)

```bash
gh pr view 123 --json title,body,commits,files,reviews,comments,statusCheckRollup
```

Show: title, description, commits, files changed, review status, CI status.

---

## Safety

- Never creates PRs from `agent-XX` branches
- Never creates PRs from `main` with no new commits
- Never approves or completes PRs, and never moves Jira issues without asking
- Warns before creating if tests failing or uncommitted changes exist
