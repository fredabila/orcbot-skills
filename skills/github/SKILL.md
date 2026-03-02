---
name: github
description: >
  Use this skill whenever an agent needs to interact with GitHub in any way.
  Covers the full GitHub workflow: setup and authentication with the GitHub CLI (gh),
  cloning repos and pushing code, creating branches, opening and reviewing PRs,
  commenting on and merging PRs, checking PR/CI status, and managing issues.
  Trigger this skill whenever the user mentions PRs, pull requests, issues, branches,
  pushing code, cloning a repo, GitHub CI, or anything GitHub-related — even casually.
  Don't try to wing GitHub tasks from memory; always consult this skill first.
---

# GitHub Skill

A complete guide for agents to interact with GitHub using the **GitHub CLI (`gh`)** and `git`.

---

## 1. Setup & Authentication

### Install the GitHub CLI

```bash
# macOS
brew install gh

# Ubuntu/Debian
sudo apt install gh

# Windows (winget)
winget install GitHub.cli

# Or via conda
conda install -c conda-forge gh
```

### Authenticate

```bash
gh auth login
# Follow the interactive prompts:
# - Choose: GitHub.com
# - Choose: HTTPS
# - Authenticate via browser (recommended) or paste a token
```

Verify auth is working:

```bash
gh auth status
```

### Configure Git identity (if not already set)

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

## 2. Cloning & Pushing Code

### Clone a repo

```bash
# By repo name (if you own it or it's an org you're part of)
gh repo clone owner/repo-name

# Standard git clone also works
git clone https://github.com/owner/repo-name.git
```

### Basic push workflow

```bash
git add .
git commit -m "feat: describe what changed"
git push origin your-branch-name
```

### Push a new branch for the first time

```bash
git push -u origin your-branch-name
```

---

## 3. Branch Management

### Create and switch to a new branch

```bash
git checkout -b feat/your-feature-name
# or
git switch -c feat/your-feature-name
```

### List branches

```bash
git branch -a        # local + remote
gh api repos/{owner}/{repo}/branches   # via API
```

### Delete a branch

```bash
git branch -d feat/your-feature-name        # local
git push origin --delete feat/your-feature-name  # remote
```

---

## 4. Pull Requests

### Create a PR with a good description

```bash
gh pr create \
  --title "feat: short summary of the change" \
  --body "## What changed\n\n- Item 1\n- Item 2\n\n## Why\n\nContext here.\n\n## Testing\n\nHow to verify." \
  --base main \
  --head feat/your-feature-name
```

**Tips for a good PR description:**
- Explain *what* changed and *why*, not just *how*
- Reference related issues: `Closes #42`
- Include testing steps
- Keep the title under 72 characters, imperative tense ("Add feature" not "Added feature")

### Open PR creation in the browser

```bash
gh pr create --web
```

### List open PRs

```bash
gh pr list
gh pr list --state all    # includes closed/merged
```

### Check PR status & CI

```bash
gh pr status               # PRs involving you
gh pr view 123             # view PR #123
gh pr checks 123           # see CI check results for PR #123
```

### Review a PR

```bash
# View the diff
gh pr diff 123

# Leave a review comment
gh pr review 123 --comment --body "Looks good, just one nit on line 42."

# Approve
gh pr review 123 --approve

# Request changes
gh pr review 123 --request-changes --body "Please fix the auth flow before merging."
```

### Merge a PR

```bash
# Merge (default)
gh pr merge 123

# Squash merge (recommended for clean history)
gh pr merge 123 --squash

# Rebase merge
gh pr merge 123 --rebase

# Merge and delete branch automatically
gh pr merge 123 --squash --delete-branch
```

### Check out a PR locally (for testing)

```bash
gh pr checkout 123
```

---

## 5. Issue Management

### Create an issue

```bash
gh issue create \
  --title "Bug: short description" \
  --body "## Steps to reproduce\n\n1. ...\n\n## Expected\n\n...\n\n## Actual\n\n..." \
  --label "bug"
```

### List issues

```bash
gh issue list
gh issue list --label "bug" --state open
```

### View an issue

```bash
gh issue view 42
```

### Comment on an issue

```bash
gh issue comment 42 --body "I can reproduce this. Working on a fix."
```

### Close an issue

```bash
gh issue close 42
gh issue close 42 --comment "Fixed in PR #123."
```

### Link issues to PRs

In the PR body, use keywords to auto-close issues on merge:
```
Closes #42
Fixes #17
Resolves #99
```

---

## 6. Best Practices

### Branch naming

| Type | Pattern | Example |
|------|---------|---------|
| Feature | `feat/short-name` | `feat/user-auth` |
| Bug fix | `fix/short-name` | `fix/login-crash` |
| Chore/refactor | `chore/short-name` | `chore/update-deps` |
| Hotfix | `hotfix/short-name` | `hotfix/prod-outage` |

### Commit messages (Conventional Commits)

```
<type>(<optional scope>): <short summary>

[optional body]

[optional footer: Closes #issue]
```

**Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`

**Examples:**
```
feat(auth): add OAuth2 login flow
fix(api): handle null response from /users endpoint
docs: update README with setup instructions
chore: bump dependencies to latest
```

### PR hygiene

- Keep PRs small and focused — one logical change per PR
- Always branch off the latest `main` (or `develop`): `git pull origin main` before branching
- Make sure CI passes before requesting review
- Respond to review comments promptly; resolve threads when addressed
- Squash or rebase before merging to keep history clean

---

## 7. Common Troubleshooting

| Problem | Fix |
|--------|-----|
| `gh: command not found` | Install via `brew install gh` or see Section 1 |
| Auth expired | Run `gh auth login` again |
| Push rejected (non-fast-forward) | Run `git pull --rebase origin main` then push again |
| Merge conflicts | `git fetch origin`, `git rebase origin/main`, resolve conflicts, `git rebase --continue` |
| Accidentally committed to main | `git checkout -b new-branch`, `git checkout main`, `git reset --hard origin/main` |
