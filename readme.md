# vibe coding is done
# Git model is pending
# AI careers is pending

# Feature
```bash
git checkout -b feat/git develop

git checkout develop
git merge -no-ff feat/git -m 'merge commit'

git branch -d feat/git
```

# Release
```bash
git checkout -b release-1.0 develop

git checkout main
git merge --no-ff release-1.2 -m 'merge commit'
git tag -a 1.2
git checkout develop
git merge --no-ff release-1.2 -m 'merge commit'

git branch -d release-1.2
```

# Hotfixes
```bash
git checkout -b hotfix-1.2.1 main

git checkout main
git merge --no-ff hotfix-1.2.1 -m 'merge commit'
git tag -a 1.2.1
git checkout develop
git merge --no-ff hotfix-1.2.1 -m 'merge commit'

git branch -d hotfix-1.2.1
```