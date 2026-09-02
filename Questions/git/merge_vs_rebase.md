# What is the difference between merge and rebase?

## Answer

Both **bring another branch’s commits into your branch**. **Merge** adds a **merge commit** and **preserves history as it happened**. **Rebase** **replays** your commits **on top of** the other branch and **rewrites commit SHAs** — linear history, no merge commit (unless you later merge).

### Mental model

You are on `feature`, `main` has moved.

```
main:    A---B---C
                \
feature:         D---E
```

**Merge** (`git checkout feature && git merge main`):

```
main:    A---B---C
                \ \
feature:         D---E---M   (M has two parents: E and C)
```

**Rebase** (`git checkout feature && git rebase main`):

```
main:    A---B---C
                      \
feature:               D'---E'   (same diffs as D,E; new hashes)
```

Then `git merge feature` into `main` is a **fast-forward** (or a PR squash).

### Comparison

| | Merge | Rebase |
| --- | --- | --- |
| History | Graph with merge commits; true “when it landed” | Linear; looks like you branched from the latest main |
| Commit hashes | Unchanged | **Rewritten** (`D` → `D'`) |
| Conflicts | Resolve **once** in the merge commit | May resolve **once per replayed commit** |
| Public/shared branches | Safe | **Dangerous** if others already pulled those commits |
| `git pull` | `git pull` (merge) | `git pull --rebase` |
| Bisect / blame | Extra merge commits | Cleaner line |
| PR on GitHub/GitLab | “Merge commit” / “squash” / “rebase and merge” are all server-side variants of this idea | Same |

### When to use which

- **Merge** `main` **into** a long-lived shared branch (`develop`, a team feature branch already on origin): no history rewrite, no force-push.
- **Rebase** your **local** (or clearly solo) feature onto latest `main` before opening/updating a PR: fewer “merge main into feature” bubbles.
- **Never rebase** commits that are **pushed and used by others** unless the team agrees and you **force-with-lease**:

```bash
git push --force-with-lease
```

(`--force-with-lease` refuses to overwrite if the remote moved unexpectedly.)

### `git pull`: merge vs rebase

```bash
git pull              # fetch + merge (merge commit if you diverged)
git pull --rebase     # fetch + rebase your local commits on top of remote
```

Config some teams use: `git config pull.rebase true`.

### Conflicts

- Merge: one conflicted tree, you commit the resolution as the merge commit.
- Rebase: conflict on commit 2 of 5 → fix → `git add` → `git rebase --continue` (or `--abort`).

### Interview one-liner

Merge **records the join**; rebase **replays your work as if it started later**. Use merge for shared history, rebase for cleaning a private branch before merge — never rewrite public history without coordination.
