# gh — GitHub CLI Discovery Guide

## Prerequisites

**gh version**: 2.11.0+ required (`gh --version` to check). Needed for `gh search code`.

**Authentication**: Run `gh auth login` first. Verify with:
```bash
gh auth status
# Shows: active account, token type, AND granted scopes
# e.g. "Token scopes: 'repo', 'read:org', 'workflow'"
```

**Scopes by use case**:
- Public repos: default scopes (no extras needed)
- Private repos: `repo`
- Org member/team endpoints: `read:org`
- Private repo Discussions (GraphQL): `read:discussion`
- Add a scope: `gh auth refresh -s <scope>`

**yq (optional)**: For YAML output. Requires v4+ (Mike Farah's rewrite).
```bash
yq --version   # v4 output contains "github.com/mikefarah/yq"
# Install from https://github.com/mikefarah/yq/ — not system package managers (may ship v3)
```
Without `yq`, all output is JSON via `--jq`. YAML output is not possible without it.

**Rate limits**:
- REST: 5,000 req/hr
- Search: 30 req/min primary. Secondary limits on burst frequency, concurrency, query cost
- GraphQL: 5,000 points/hr (scales with node count — use `first: N` to control)
- Headers only visible via `gh api --include` or `gh api --verbose`

---

## Discovery Patterns

### Conventions

**Placeholders**: `OWNER`, `REPO`, `ORG`, `LOGIN` are values you substitute. `gh api` does not expand template syntax.

**`--` separator**: Required before query strings containing operators like `NOT` or `created:>=` to prevent them from being parsed as flags.

**Platform dates**: GNU/Linux: `$(date -d '7 days ago' +%Y-%m-%d)`, macOS: `$(date -v-7d +%Y-%m-%d)`

### Native CLI

#### 1. Repo Discovery
```bash
# Combined filters with result-count control
gh search repos --language typescript --owner anthropics --sort stars --limit 5

# List repos in an org (works for users too — no --sort flag, default order)
gh repo list anthropics --limit 20

# Repo metadata
gh repo view cli/cli --json description,stargazerCount,topics
```

#### 2. Code Discovery

The primary use case: you've already exhausted local options and need to find a solution out in the world. Start broad — don't limit to your target language. Algorithms and patterns port. A working implementation in any language is often enough to adapt.

```bash
# Find ANY implementation of what you need — language doesn't matter yet
gh search code "topological sort"

# Narrow to languages with strong ecosystem for this domain
gh search code "topological sort" --language python
gh search code "topological sort" --language rust

# Only after you've found the approach, look for one in your target language
gh search code "topological sort" --language typescript
```

Noise reduction and scoping:
```bash
# Exclude vendor/dependency directories (Code Search v2 query syntax)
gh search code "serve NOT path:vendor NOT path:node_modules" --repo anthropics/sdk

# Filter by filename (--filename is an inclusion flag; no exclusion flags exist)
gh search code "import" --filename "mod.ts" --language typescript

# Scope to an org
gh search code "pattern" --owner anthropics
```

#### 3. Issue/PR Discovery
```bash
# Search issues with filters
gh search issues "bug" --repo cli/cli --sort comments --label bug

# List open issues in a repo
gh issue list --repo cli/cli --label bug --state open

# Search merged PRs by author
gh search prs --author LOGIN --repo OWNER/REPO --merged --sort updated

# Issue velocity — count recent issues to gauge activity
SINCE=$(date -d '7 days ago' +%Y-%m-%d)   # macOS: $(date -v-7d +%Y-%m-%d)
gh search issues --json number --repo OWNER/REPO -- "created:>=$SINCE" | jq length
```

#### 4. Release Discovery
```bash
# Recent releases
gh release list --repo OWNER/REPO --limit 10

# Latest release details
gh release view --repo OWNER/REPO

# Specific version
gh release view v1.2.3 --repo OWNER/REPO
```

#### 5. Trend Discovery
There is no CLI equivalent to GitHub's trending page. Approximate with date-filtered search:
```bash
# Repos created in last 7 days, sorted by stars (approximation)
gh search repos --sort stars -- "created:>=$(date -d '7 days ago' +%Y-%m-%d)"
# macOS: gh search repos --sort stars -- "created:>=$(date -v-7d +%Y-%m-%d)"

# Recently updated repos in a language
gh search repos --language rust --sort updated -- "pushed:>=$(date -d '30 days ago' +%Y-%m-%d)"
```

### Requires `gh api` (REST)

#### 6. People/Activity Discovery
There is no `gh search people`. Use REST:
```bash
# Top 3 non-anonymous contributors by contribution count
gh api repos/OWNER/REPO/contributors \
  --jq '[.[] | select(.login != null)] | sort_by(-.contributions)[0:3] | .[].login'

# Then for each login — their recent merged PRs in that repo
gh search prs --author LOGIN --repo OWNER/REPO --merged --sort updated

# Recent repo activity events
gh api repos/OWNER/REPO/events --jq '.[:5] | .[] | "\(.type) by \(.actor.login)"'
```

#### 7. Organization Discovery
```bash
# List org repos (--limit default is 30, no hard cap)
gh repo list ORG --limit 100

# Full enumeration via API (follows Link headers to exhaustion)
gh api orgs/ORG/repos --paginate --jq '.[].full_name'

# NOTE: /orgs/ only works for organizations.
# For user accounts: gh api users/USERNAME/repos --paginate --jq '.[].full_name'
# gh repo list works for both (abstracts the distinction)
```

### Requires `gh api graphql`

#### 8. Discussion Discovery
`gh` has no native Discussion subcommand. Use GraphQL:
```bash
# List discussions in a repo
gh api graphql -F owner=OWNER -F name=REPO -f query='
  query($owner: String!, $name: String!) {
    repository(owner: $owner, name: $name) {
      discussions(first: 10) {
        nodes {
          id
          title
          body
          answer { body author { login } }
          comments(first: 5) {
            nodes { body author { login } }
          }
        }
        pageInfo { endCursor hasNextPage }
      }
    }
  }
'

# For pagination, see Pattern 9 below
```

### API Escape Hatch

#### 9. When to Use `gh api`
Fall back to `gh api` when:
- Data isn't exposed by a `gh` subcommand (Discussions, contributors, org members)
- You need full pagination beyond `--limit`
- You need nested/related data in one query (GraphQL)
- You need response headers (rate limit status)

**REST pagination**:
```bash
# --paginate follows Link headers automatically (no result cap on non-search endpoints)
gh api repos/OWNER/REPO/issues --paginate --jq '.[].title'
```

**GraphQL pagination** (cursor-based):
```bash
# Initial query — get first page + cursor
CURSOR=$(gh api graphql -F owner=OWNER -F name=REPO -f query='
  query($owner: String!, $name: String!) {
    repository(owner: $owner, name: $name) {
      discussions(first: 10) {
        pageInfo { endCursor hasNextPage }
      }
    }
  }
' --jq '.data.repository.discussions.pageInfo.endCursor')

# Follow-up — query must declare $after variable and use it
gh api graphql -F owner=OWNER -F name=REPO -f after="$CURSOR" -f query='
  query($owner: String!, $name: String!, $after: String) {
    repository(owner: $owner, name: $name) {
      discussions(first: 10, after: $after) {
        nodes { id title }
        pageInfo { endCursor hasNextPage }
      }
    }
  }
'
```

**Header inspection**:
```bash
# --include prints response headers before body
gh api repos/OWNER/REPO --include

# --verbose prints full request AND response
gh api repos/OWNER/REPO --verbose
```

---

## Social Framing

GitHub's primitives map to social concepts, with caveats:

| Primitive | Analogy | Where it breaks |
|-----------|---------|-----------------|
| Issues | Threaded comments + reactions | No voting/ranking; chronological order |
| Discussions | Forum topics with upvotes + answers | Upvotes are on comments, not threads. "Accepted answer" has no social parallel |
| PRs | Review threads with inline discussion | Technical review, not general discussion |
| Stars | Bookmarks | No algorithmic effect; purely a count |
| Follows | Dashboard feed subscriptions | Shows public activity in dashboard, NOT notifications |
| Watch | Notification subscriptions | Opt-in per-repo. Generates notifications (unlike Follows) |

---

## Interaction Patterns

All commands include `--repo` for agents without a local clone:

```bash
# Issues
gh issue create --repo OWNER/REPO --title "Title" --body "Body"
gh issue comment 123 --repo OWNER/REPO --body "Comment"
gh issue close 123 --repo OWNER/REPO
gh issue edit 123 --repo OWNER/REPO --add-label "bug"

# PRs (--head/--base required without local git state)
gh pr create --repo OWNER/REPO --head BRANCH --base main --title "Title" --body "Body"
gh pr review 123 --repo OWNER/REPO --approve
gh pr review 123 --repo OWNER/REPO --comment --body "Feedback"
gh pr review 123 --repo OWNER/REPO --request-changes --body "Fix X"
gh pr merge 123 --repo OWNER/REPO --squash
# NOTE: Self-approval blocked — authors cannot approve their own PRs (HTTP 422)

# Starring (no native gh repo star — use API)
gh api user/starred/OWNER/REPO -X PUT     # star
gh api user/starred/OWNER/REPO -X DELETE  # unstar

# Forking
gh repo fork OWNER/REPO --clone
```

---

## Output Formatting

**`--jq`** (built-in, always available) — filters JSON output:
```bash
# Regular gh subcommands need --json to switch to JSON mode
gh issue list --repo cli/cli --json number,title,state --jq '.[] | "\(.number): \(.title)"'

# gh api already outputs JSON — use --jq directly (no --json flag)
gh api repos/cli/cli --jq '.stargazers_count'
```

**`yq` (optional, v4+)** — converts JSON to YAML:
```bash
# gh subcommand → JSON → YAML
gh repo view cli/cli --json name,description | yq -p json -o yaml

# gh api → YAML
gh api repos/cli/cli | yq -p json -o yaml
```

**Decision rule**: `--jq` always works for filtering. Add `| yq -p json -o yaml` when YAML output is needed and `yq` is available.

---

## Error Handling

| Failure | Symptom | Response |
|---------|---------|----------|
| Not authenticated | `gh auth status` shows no account | `gh auth login` |
| Missing scope | HTTP 403; private repos return 404 (indistinguishable from not-found) | `gh auth status` to check scopes, `gh auth refresh -s SCOPE` to add |
| Rate limited (REST) | HTTP 403, `X-RateLimit-Remaining: 0` (via `--include`) | Check `X-RateLimit-Reset` header for UNIX reset timestamp |
| Rate limited (secondary) | HTTP 403/429 with `Retry-After` header (via `--include`) | Wait the number of seconds in `Retry-After` |
| Rate limited (GraphQL) | HTTP 200, `"type": "RATE_LIMITED"` in response body | Add `first: N` limits to reduce point cost |
| GraphQL error | HTTP 200 with `"errors": [...]` in body | Read error message. GraphQL always returns 200 — never trust HTTP status for success |
| Repo not found | HTTP 404 | Check spelling. Also check `gh auth status` for `repo` scope (private repos return 404 without it) |
| Search: 0 results | Empty set | Remove qualifiers one at a time, try substrings, broaden via `gh api` |
| Search: 1000 cap | Search API caps at 1000 total results | Narrow query. Non-search `--paginate` has no cap |
| Self-approval | HTTP 422 on `--approve` | PR authors can't approve their own PRs |
| Org/user mismatch | HTTP 404 on `/orgs/NAME/repos` | Use `/users/NAME/repos` for user accounts |
