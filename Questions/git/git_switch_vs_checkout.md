# What is the difference between `git switch` and `git checkout`?

## Answer

`git checkout` is the **old, overloaded** command (branches **and** files). `git switch` (Git 2.23+) does **only branch switching**. Same result for “move HEAD to another branch”; different scope and fewer accidents.

### Side by side

| | `git switch` | `git checkout` |
| --- | --- | --- |
| Purpose | Change branches (or detach HEAD) | Branches **or** restore files **or** detach HEAD |
| Added | Git 2.23 (2019) | Original Git |
| Restore a file | No — use `git restore` | `git checkout -- file` (easy to mix up with branch checkout) |
| New branch | `git switch -c feature` | `git checkout -b feature` |
| Existing branch | `git switch main` | `git checkout main` |
| Detached HEAD | `git switch --detach <commit>` | `git checkout <commit>` |
| Safety | Refuses some ambiguous cases; designed to not touch the index/worktree the way file-checkout does | Easy to run `git checkout file` thinking it was a branch |

### Equivalent pairs

```bash
git checkout main
git switch main

git checkout -b feature
git switch -c feature

git checkout --detach abc123
git switch --detach abc123
```

**Not** equivalent — restoring files:

```bash
git checkout -- app.js      # old way: throw away local edits to app.js
git restore app.js          # new way (pair of git switch)

git checkout main -- app.js # take app.js from branch main
git restore --source=main -- app.js
```

### Uncommitted changes

Both **refuse to switch** if the switch would **overwrite** local modifications. Stash, commit, or carry changes that don’t conflict. `git switch` will not silently clobber files the way people fear from misusing `git checkout --`.

### What to say in an interview

- `checkout` = Swiss Army knife (branch + paths + commits).
- `switch` + `restore` split that into two clear tools.
- Prefer `git switch` / `git restore` on modern Git; `checkout` still works everywhere and shows up in old docs/scripts.
