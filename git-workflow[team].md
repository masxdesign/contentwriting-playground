Perfect 🚀 Let’s document your **team workflow** using the custom Git commands (aliases) we built into `.gitconfig`. This will give you a clean Git Flow–style process that’s simple and repeatable.

---

# 📚 Git Workflow with Custom Commands

We follow [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/) with simplified custom commands for **features**, **releases**, **hotfixes**, and **cleanup**.

---

## 🔀 Branching Model

* **`main`** → production-ready code.
* **`develop`** → integration branch, where features are merged.
* **`feature/*`** → short-lived branches for new work.
* **`release/*`** → preparation for a new production release.
* **`hotfix/*`** → urgent production fixes.

---

## ✨ Feature Workflow

Start a new feature from `develop`:

```bash
git feature-start login-ui
```

Finish a feature (merge into `develop`, delete local + remote branch, push updates):

```bash
git feature-finish login-ui
```

---

## 🚀 Release Workflow

Start a release branch from `develop`:

```bash
git release-start 1.0
```

Finish a release (merge into `main`, tag release, merge back into `develop`, cleanup branches, push):

```bash
git release-finish 1.0
```

---

## 🩹 Hotfix Workflow

Start a hotfix branch from `main`:

```bash
git hotfix-start 1.0.1
```

Finish a hotfix (merge into `main`, tag hotfix, merge back into `develop`, cleanup branches, push):

```bash
git hotfix-finish 1.0.1
```

---

## 🧹 Cleanup

Remove local branches already merged into `develop` or `main`:

```bash
git cleanup
```

Remove stale remote-tracking branches (prune deleted branches on GitHub):

```bash
git cleanup-remote
```

---

## ✅ Example Workflow in Practice

### 1. Start a feature

```bash
git feature-start user-auth
```

### 2. Work, commit, push if needed

```bash
git push -u origin feature/user-auth
```

### 3. Finish the feature

```bash
git feature-finish user-auth
```

### 4. Prep a release

```bash
git release-start 1.1
```

... finalize version, docs, changelog ...

```bash
git release-finish 1.1
```

### 5. Emergency fix

```bash
git hotfix-start 1.1.1
# fix bug
git hotfix-finish 1.1.1
```

### 6. Regular maintenance

```bash
git cleanup
git cleanup-remote
```

---

👉 This gives you a **repeatable workflow** with just a few custom commands instead of long manual Git sequences.

Would you like me to also create a **one-page Markdown cheatsheet** (`WORKFLOW.md`) for your repo, so your whole team can follow this workflow consistently?
