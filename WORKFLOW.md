# Git Workflow (Git Flow with Custom Commands)

This repository follows the [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/) model with a few helper aliases to make everyday operations fast and consistent.

---

## Branching Model

- **main** — production-ready code.
- **develop** — integration branch; features merge here.
- **feature/*** — short-lived branches created from `develop`; merged back into `develop`.
- **release/*** — preparation branches created from `develop`; merged into **main** and back into **develop**.
- **hotfix/*** — urgent fixes created from **main**; merged into **main** and back into **develop**.

---

## Required Git Aliases

Add these under the `[alias]` section of your `~/.gitconfig`:

```ini
[alias]
	ai-commit = "!~/.git-ai-commit.sh"
	
	# Common shortcuts
	co = checkout
	cb = checkout -b
	br = branch
	st = status -sb
	lg = log --oneline --graph --decorate --all

	# --- Feature ---
	feature-start = "!f() { git checkout develop && git checkout -b feature/$1; }; f"
	feature-finish = "!f() { \
		git checkout develop && \
		git merge --no-ff feature/$1 -m \"Merge feature $1\" && \
		git branch -d feature/$1 && \
		git push origin develop && \
		git push origin --delete feature/$1; \
	}; f"

	# --- Release ---
	release-start = "!f() { git checkout develop && git checkout -b release-$1; }; f"
	release-finish = "!f() { \
		if git show-ref --verify --quiet refs/heads/main; then default=main; else default=master; fi; \
		git checkout $default && \
		git merge --no-ff release-$1 -m \"Release $1\" && \
		git tag -a $1 -m \"Release $1\" && \
		git checkout develop && \
		git merge --no-ff release-$1 -m \"Merge release $1 back into develop\" && \
		git branch -d release-$1 && \
		git push origin $default --tags && \
		git push origin develop; \
		if git ls-remote --exit-code origin release-$1 >/dev/null 2>&1; then \
		git push origin --delete release-$1; \
		else \
		echo 'ℹ️ Remote branch release-$1 already deleted'; \
		fi; \
	}; f"

	# --- Hotfix ---
	hotfix-start = "!f() { \
		if git show-ref --verify --quiet refs/heads/main; then default=main; else default=master; fi; \
		git checkout $default && \
		git checkout -b hotfix-$1; \
  	}; f"
	hotfix-finish = "!f() { \
		if git show-ref --verify --quiet refs/heads/main; then default=main; else default=master; fi; \
		git checkout $default && \
		git merge --no-ff hotfix-$1 -m \"Hotfix $1\" && \
		git tag -a $1 -m \"Hotfix $1\" && \
		git checkout develop && \
		git merge --no-ff hotfix-$1 -m \"Merge hotfix $1 into develop\" && \
		git branch -d hotfix-$1 && \
		git push origin $default --tags && \
		git push origin develop; \
		if git ls-remote --exit-code origin hotfix-$1 >/dev/null 2>&1; then \
		git push origin --delete hotfix-$1; \
		else \
		echo 'ℹ️ Remote branch hotfix-$1 already deleted'; \
		fi; \
	}; f"

	# --- Update current feature with latest develop (rebase) ---
	feature-update = "!f() { \
		current=$(git rev-parse --abbrev-ref HEAD); \
		if [[ $current != feature/* ]]; then \
		echo '⚠️ Not on a feature branch (feature/*)'; exit 1; \
		fi; \
		git fetch origin develop && \
		git checkout develop && git pull origin develop && \
		git checkout $current && \
		git rebase develop; \
	}; f"

	# --- Cleanup ---
	cleanup = "!git branch --merged | egrep -v '(^\\*|master|main|develop)' | xargs -n 1 git branch -d"
	cleanup-remote = "!git fetch -p && git remote prune origin"
```

> **Note:** The `feature-finish`, `release-finish`, and `hotfix-finish` aliases push updates and delete the remote branch by design.

---

## Daily Commands (Cheat Sheet)

### Start a feature
```bash
git feature-start user-auth
# (optional) push the branch the first time:
git push -u origin feature/user-auth
```

### Finish a feature (merge → develop, push, delete feature)
```bash
git feature-finish user-auth
```

### Start a release
```bash
git release-start 1.1
```

### Finish a release (merge → main, tag, merge back → develop, push, delete branch)
```bash
git release-finish 1.1
```

### Start a hotfix
```bash
git hotfix-start 1.1.1
```

### Finish a hotfix (merge → main, tag, merge back → develop, push, delete branch)
```bash
git hotfix-finish 1.1.1
```

### Update current feature with latest `develop` (rebases)
```bash
git feature-update
```

### Clean merged local branches (except main/develop)
```bash
git cleanup
```

### Prune stale remote branches
```bash
git cleanup-remote
```

---

## Handling Urgent Fixes While a Feature Is In Progress

When a hotfix is needed on `main` while you’re working on a feature:

1. Run the hotfix flow (this also merges the fix back into `develop`):
   ```bash
   git hotfix-start 1.2.1
   # implement fix, commit
   git hotfix-finish 1.2.1
   ```

2. Update your feature with the latest `develop`:
   ```bash
   git checkout feature/your-branch
   git feature-update  # rebases onto updated develop
   ```

3. Resolve any conflicts and continue:
   ```bash
   git rebase --continue
   ```

---

## Rebase vs Merge (Keeping Your Feature Up-to-Date)

### Rebase (Preferred for solo features)
```bash
git feature-update
```
**Pros:** linear, clean history; easier reviews.  
**Cons:** avoid rebasing shared branches (force-push required).

### Merge (Preferred for shared feature branches)
```bash
git checkout feature/your-branch
git pull origin develop
git merge develop
```
**Pros:** safe for collaborative branches; preserves history.  
**Cons:** more merge commits; noisier history.

**Team Rule of Thumb**
- Solo feature → **rebase** with `git feature-update`.
- Shared feature → **merge** `develop` regularly.

---

## Tagging Guidance

- Use **annotated tags** for releases/hotfixes:
  ```bash
  git tag -a 1.2 -m "Release 1.2"
  git push origin 1.2
  ```
- Prefer semantic versioning (e.g., `v1.2.0`) if your tooling expects it.

---

## Branch Protections (Recommended)

Configure in GitHub → **Settings → Branches**:

- Protect **main** and **develop**:
  - Require pull requests.
  - Require status checks to pass (CI).
  - Dismiss stale reviews on new commits.
  - Restrict who can push and who can dismiss checks.
  - Prevent force pushes and deletions.

---

## Troubleshooting

- **“Not on a feature branch”** when running `git feature-update`:  
  Make sure your current branch name matches `feature/*`.

- **Deletion refused with `-d`**:  
  Branch isn’t fully merged. Use `-D` **only if** you’re certain it’s safe.

- **Remote still shows deleted branches**:  
  Run `git cleanup-remote` to prune stale refs.

---

_This document is the canonical reference for branching/release practices in this repo. Keep it in sync with team norms and tooling._


hello