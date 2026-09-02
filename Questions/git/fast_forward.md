# What is a fast-forward in Git?

## Answer

A **fast-forward** is a merge where Git **does not create a merge commit**. It just **moves the branch pointer** forward, because the branch you are merging **into** has **no unique commits** — its tip is a **direct ancestor** of the other branch.

### When it can happen

```
main:     A---B
               \
feature:        C---D
```

`main` has not moved since you branched. Merge `feature` into `main`:

```bash
git checkout main
git merge feature
```

Git **fast-forwards** `main` from `B` to `D`:

```
main, feature:  A---B---C---D
```

No merge commit. History stays **linear**. Same graph as if those commits had been made on `main`.

### When it cannot happen (true merge)

```
main:     A---B---E
               \
feature:        C---D
```

`main` grew `E` while you worked. Fast-forward is **impossible**. Default `git merge` creates a **merge commit** `M` with two parents (`E` and `D`):

```
main:     A---B---E---M
               \     /
feature:        C---D
```

That is a **non-fast-forward** / **3-way merge**.

### Commands interviewers expect

```bash
git merge feature              # fast-forward if possible, else merge commit
git merge --ff-only feature    # succeed only if fast-forward; otherwise abort
git merge --no-ff feature      # always create a merge commit, even if FF is possible
```

`--no-ff` is what many teams use on `main` so every feature shows as a **merge bubble** (easy to revert a whole PR with one commit).

`--ff-only` is what you want on a **release** or **deploy** branch: “only move the pointer; never invent a merge commit.”

### Fast-forward vs rebase vs merge commit

| | Fast-forward | Merge commit (`--no-ff` or diverged) | Rebase then FF |
| --- | --- | --- | --- |
| New commit? | No — pointer only | Yes — 2 parents | Rebase rewrites feature commits, then merge can FF |
| History | Linear | Graph / “bubbles” | Linear (after rebase) |
| Condition | Target is ancestor | Histories diverged, or you forced `--no-ff` | You replayed first |

Rebasing `feature` onto `main` **makes a later merge fast-forwardable**, because `feature` then sits **on top of** `main`.

### `git pull` and “non-fast-forward”

- `git pull` = fetch + merge. If your local `main` **diverged** from `origin/main`, the merge is **not** a fast-forward.
- `git push` rejected as **non-fast-forward** means the **remote** has commits you don’t have. Fetch/rebase or pull first; don’t `--force` shared branches.

```bash
git pull --ff-only     # fail instead of creating a local merge commit
```

### Interview one-liner

Fast-forward = **slide the branch label forward** because there is nothing to join. If both sides have unique commits, Git **cannot** fast-forward and must create a **merge commit** (unless you rebase first).


Suppose we have commits `A → B → C`, and `C` is where the `main` branch currently points.

We create a `feature` branch from `main` and add two commits, `X` and `Y`:

```text
A → B → C → X → Y
        ↑         ↑
       main     feature
```

Here, `C` is an ancestor of the `feature` branch (`Y`).

If `main` has not made any additional commits after `feature` was created, then `main` is simply behind `feature`. Therefore, when we merge `feature` into `main`, Git does not need to create a new merge commit.

It can simply move the `main` pointer forward from `C` to `Y`. This is called a **fast-forward merge**.

```text
A → B → C → X → Y
                    ↑
               main, feature
```

However, if `main` makes another independent commit after the `feature` branch was created:

```text
          X → Y
         /
A → B → C
         \
          Z
          ↑
        main
```

Now the branches have **diverged**.

`Z` is not an ancestor of `Y`, and `Y` is not an ancestor of `Z`. Therefore, Git cannot simply move the `main` pointer to `Y`, so a normal merge will create a new **merge commit**.

One important distinction is that **fast-forward and rebase are not the same thing**.

A fast-forward does not modify or recreate any commits; it simply moves a branch pointer forward.

A rebase, on the other hand, replays commits onto a different base and creates new commit objects. Rebase can, however, make a subsequent merge possible as a fast-forward.


git merge feature	Default — FF if possible, otherwise merge commit
git merge --ff feature	Explicitly same general behavior
git merge --no-ff feature	Always create merge commit
git merge --ff-only feature	FF or fail

