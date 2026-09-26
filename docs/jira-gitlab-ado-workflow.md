# Jira + GitLab + Azure DevOps workflow

Adapting the `/issue → /take → /review → /pr → /merge` commands for a client
that tracks issues in **Jira Cloud** and hosts code in **self-hosted GitLab**
and **Azure DevOps Services (cloud)**.

Approach **B**: one set of commands. The forge is auto-detected from
`git remote get-url origin`; the tracker is declared per repo (manually, since
one Jira instance spans many GitLab and ADO repos).

Status legend: `[ ]` todo · `[~]` in progress · `[x]` done

---

## 1. Tools to install

| Status | Tool | Purpose | Install |
|---|---|---|---|
| [x] | `acli` | Atlassian CLI (official, Jira Cloud) | `brew tap atlassian/homebrew-acli && brew install acli` |
| [x] | `glab` | GitLab CLI (MRs, self-hosted) | `brew install glab` |
| [x] | `azure-cli` | Azure CLI | `brew install azure-cli` |
| [x] | `azure-devops` ext | `az repos` / `az devops` subcommands | `az extension add --name azure-devops` |
| [x] | `direnv` | Per-client env vars (hosts, tokens) | `brew install direnv` + `eval "$(direnv hook zsh)"` |
| [x] | `jq` | Parse `--json` output in commands | `brew install jq` (likely already present) |

Installed 2026-09-26: acli 1.3.39, glab 1.119.0, azure-cli 2.90.0, azure-devops 1.0.8, direnv 2.37.1, jq 1.8.2.
`acli jira workitem` subcommands verified (`view --json`, `search`, `transition`, `assign`).

| Status | Task |
|---|---|
| [x] | Add `eval "$(direnv hook zsh)"` to zsh config (binary installed, hook not set up) |

Optional: add the brew packages to `claude/brew_packages` once the setup is proven.

## 2. Credentials and setup

| Status | Task | Notes |
|---|---|---|
| [x] | Create Jira Cloud API token | id.atlassian.com → Security → API tokens |
| [x] | `acli jira auth login` | `--site <site>.atlassian.net --email <email> --token` (token via stdin) |
| [x] | Create GitLab PAT | Scopes: `api`, `read_repository`, `write_repository` |
| [x] | `glab auth login --hostname <gitlab-host>` | Token stored in OS keyring (not plaintext config) |
| [x] | ADO auth | Using `az login` (Entra, short-lived) instead of a PAT. Conditional access forces re-login every 12h (AADSTS70043). Decide: keep `az login` or switch to PAT (`az devops login`) |
| [x] | `az devops configure --defaults organization=https://dev.azure.com/<org>` | Project is derived per repo |
| [ ] | Store tokens in macOS keychain | `security add-generic-password -s <name> -a "$USER" -w` |
| [ ] | Client `.envrc` in `~/work/<client>/` | Export `AZURE_DEVOPS_EXT_PAT`, `GITLAB_HOST`, etc. from keychain; keep out of dotfiles |
| [x] | Verify each CLI | `acli jira workitem view <KEY-1>`, `glab mr list`, `az repos pr list` |
| [~] | Run `/harden` | Re-run 2026-09-26 after login: PASS, but audit doesn't model Read tool, `gh`/`git push` sends, `echo $VAR`, or glab/acli token stores (see /harden tickets). acli token location unknown |
| [ ] | Decide attribution on Jira / MR comments | Check client policy on AI-generated content |

## 3. Per-repo config

Each repo declares its tracker manually. Proposed location: `CLAUDE.local.md`
(personal, not committed to the client's repo).

```markdown
## Workflow
- tracker: jira
- jira-site: <site>.atlassian.net
- jira-project: ABC
- transitions: start="In Progress", review="Acceptatie"
```

Forge detection from `origin`:

| Remote matches | Forge | CLI |
|---|---|---|
| `github.com` | GitHub | `gh` |
| `dev.azure.com` / `*.visualstudio.com` | Azure DevOps | `az repos` |
| anything else | GitLab | `glab` |

| Status | Task |
|---|---|
| [x] | Confirm `CLAUDE.local.md` as config location (vs committed `.claude/workflow.md`) |
| [x] | Confirm Jira transition names used by the client's project — `To Do`, `In Progress`, `waiting`, `Acceptatie`, `Done` (exact casing; also `Refinement status`, `Won't Do`). `/take` uses `start="In Progress"` |
| [x] | `/pr` moves the issue to `Acceptatie` (`review="Acceptatie"`) |
| [ ] | Decide which status (if any) to move to after merge |
| [x] | Check `CLAUDE.local.md` is gitignored in each client repo before adding it (user handles per repo) |

## 4. Known constraints

- **Branch names**: `feat/ABC-123-short-desc` — Jira key, no `#`. Key in branch
  and commit message links them in Jira's development panel.
- **ADO**: no ADO work items, no work-item linking policy. Every PR requires
  review and manual approval from another reviewer — `/merge` cannot
  self-approve; it must report reviewer/policy status and at most set
  auto-complete (`az repos pr update --auto-complete true`).
- **Jira Cloud** descriptions are ADF; verify `## Prompt Contract` parsing
  still works on rendered output.
- **Epics**: use Jira parent/child (`acli jira workitem search --jql "parent = ABC-10"`)
  instead of parsing task lists.

## 5. Command adaptation

| Status | Command | Changes |
|---|---|---|
| [~] | `/take` | Written 2026-09-26, read-only queries verified; needs a real run on an issue. Jira fetch, assign + transition to In Progress, Jira-key branch names, Epic Mode via parent/child |
| [ ] | `/pr` | `glab mr create` / `az repos pr create`; Jira key in title; transition to In Review |
| [ ] | `/review` | Fetch MR/PR diff via `glab` / `az repos` |
| [ ] | `/merge` | GitLab: `glab mr merge`; ADO: check approvals, auto-complete only |
| [ ] | `/issue` | List/create Jira work items via JQL |
| [ ] | `/purge` | Remote branch cleanup on GitLab / ADO |
