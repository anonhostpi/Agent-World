# gh — GitHub CLI Discovery Guide

This guide is for agents using the GitHub CLI (`gh`) to discover, explore, and interact with the GitHub ecosystem. It focuses on **discovery patterns** — how to find things, not just how to run commands.

---

## Discovery Patterns

### Finding Repos

```bash
# Search repos by topic/keyword
gh search repos "accessibility cli" --sort stars
gh search repos "cli tool" --language typescript --sort stars

# Search within a specific org
gh search repos "mcp" --owner anthropics

# Get repo metadata
gh repo view owner/repo
gh repo view owner/repo --json description,stargazerCount,languages,topics

# List repos for a user/org
gh repo list anthropics --sort stars --limit 20
gh repo list anthropics --language typescript
```

### Finding Code

```bash
# Search code across all of GitHub
gh search code "pattern" --language typescript
gh search code "import { serve }" --filename "mod.ts"

# Narrow to a repo or org
gh search code "pattern" --repo owner/repo
gh search code "pattern" --owner anthropics
```

### Finding People and Activity

```bash
# Who's active on a repo?
gh api repos/owner/repo/contributors --jq '.[].login'

# Recent activity on a repo
gh api repos/owner/repo/events --jq '.[] | "\(.type) by \(.actor.login) at \(.created_at)"'

# What's someone working on?
gh search prs --author username --state open
gh search issues --author username --state open
```

### Exploring Issues and Discussions

```bash
# Search issues across GitHub
gh search issues "bug" --repo owner/repo --sort comments
gh search issues "feature request" --label enhancement --repo owner/repo

# List issues with filters
gh issue list --repo owner/repo --label bug --state open
gh issue list --repo owner/repo --assignee @me

# Read an issue
gh issue view 123 --repo owner/repo
gh issue view 123 --repo owner/repo --comments
```

### Exploring PRs

```bash
# Search PRs
gh search prs "refactor" --repo owner/repo --state merged
gh pr list --repo owner/repo --state open --limit 20

# Read a PR with full context
gh pr view 123 --repo owner/repo
gh pr view 123 --repo owner/repo --comments
gh pr diff 123 --repo owner/repo

# Review comments (not captured by --comments)
gh api repos/owner/repo/pulls/123/comments
```

### Exploring Releases and Tags

```bash
# Latest release
gh release view --repo owner/repo

# All releases
gh release list --repo owner/repo --limit 10

# Release notes for a specific version
gh release view v1.2.3 --repo owner/repo
```

### Exploring the API Directly

```bash
# Any GitHub REST API endpoint
gh api repos/owner/repo
gh api repos/owner/repo/readme --jq '.content' | base64 -d

# GraphQL for complex queries
gh api graphql -f query='
  query {
    repository(owner: "owner", name: "repo") {
      discussions(first: 10) {
        nodes { title body }
      }
    }
  }
'

# Paginated results
gh api repos/owner/repo/issues --paginate --jq '.[].title'
```

---

## Interaction Patterns

### Issues

```bash
# Create
gh issue create --repo owner/repo --title "Title" --body "Body"

# Comment
gh issue comment 123 --repo owner/repo --body "Comment"

# Close
gh issue close 123 --repo owner/repo

# Label
gh issue edit 123 --repo owner/repo --add-label "bug"
```

### PRs

```bash
# Create from current branch
gh pr create --title "Title" --body "Body"

# Review
gh pr review 123 --approve
gh pr review 123 --comment --body "Feedback"
gh pr review 123 --request-changes --body "Fix X"

# Merge
gh pr merge 123 --squash --delete-branch
```

### Starring and Forking

```bash
gh repo fork owner/repo --clone
gh api user/starred/owner/repo -X PUT   # star
gh api user/starred/owner/repo -X DELETE # unstar
```

---

## Output Formatting

```bash
# JSON output for structured parsing
gh repo view owner/repo --json name,description,url

# JQ filtering
gh issue list --json number,title,labels --jq '.[] | "\(.number): \(.title)"'

# YAML-like output (use --json + yq if available)
gh repo view owner/repo --json name,description | yq -p json -o yaml
```

---

## Discovery Workflow

When exploring something new on GitHub:

1. **Start broad**: `gh search repos` to find relevant projects
2. **Read the README**: `gh api repos/owner/repo/readme --jq '.content' | base64 -d`
3. **Check activity**: recent issues, PRs, releases — is the project alive?
4. **Read the issues**: understand what problems people hit and what's planned
5. **Read the code**: `gh search code` to find specific patterns or implementations
6. **Check discussions**: `gh api` with GraphQL for community conversations
