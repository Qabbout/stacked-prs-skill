# Phase C: Land the stack

Every action on the host here (merging a PR, changing a target, closing a PR, deleting a branch) and every tracker change needs the user's authorization for that scope; one authorization may cover a concrete batch. Every push needs an approved `safe-push.sh` plan (rule 4).

## 0. Retargeting: let the host do it, and check every time

When a merge deletes the merged PR's branch, some hosts move every open PR that targeted it to the merged PR's target. In `a ← b ← c`, merging `b` into `a` makes `c` target `a`.

| Host | Retargets when a merge deletes the branch? | Do |
|---|---|---|
| GitHub | Yes ([documented](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/merging-a-pull-request)) | Merge with "delete branch" on |
| Bitbucket | Unverified; confirm current behavior for this repository | Retarget dependents before deleting the parent unless verified |
| GitLab, others | Check current documentation or an authorized test, then record it in the stack map | If unverified, retarget by hand before merging |

- **Before merging**, note the open PRs that target the branch about to merge (the stack map's "Other PRs target it?" column).
- **Right after merging**, check each of those PRs:
  - **open, and targets the merged PR's target:** continue with step 3.
  - **open, but still targets the deleted branch:** retarget it by hand now (with approval), and tell the user.
  - **closed:** stop and tell the user, then follow `changes.md`, "A slice PR was closed".
- **No merge, no retarget.** An integration branch, or a slice dropped without merging, is never merged through a PR. Retarget its dependents by hand (with approval) before the branch goes away. Never delete a branch by hand while an open PR targets it.
- **The reference PR is closed only when the user decides to.**

## 1. Know how the host merges

```bash
git log --oneline --first-parent -10 origin/<base>
```

- **Merge commit** ("Merge pull request #N" or "Merged in …"): the PR's commits stay in the base's history. The PRs above it only need retargeting and a merge of the base.
- **Squash or rebase merge** (one new commit per PR): the PR's original commits don't exist on the base. The PRs above it still contain them, so their diffs show the landed changes again until step 3 fixes them.

If you can choose, a merge commit is better for a stack. Squash is fine for the last PR, or for a slice nothing else depends on.

## 2. Merge order

- A PR merges only after every PR it depends on has merged.
- Independent slices that target the base can merge in any order once they're approved.
- Before a PR merges, check that its diff against its target is only its own slice (`git diff --stat origin/<target>...origin/<branch>`), and that CI is green.

## 3. After each PR lands

```bash
git fetch origin
$SKILL/scripts/stack-status.sh <branch>:<target> …
```

1. **PRs whose target was the merged branch:** confirm step 0's retarget happened. On a host that doesn't retarget, retarget each one to the base by hand, after approval.
2. **Bring them up to date** by merging the base: `git merge --no-ff -m "Merge <base> into <branch>" origin/<base>`.
   - After a merge-commit landing, shared ancestry is preserved; newer base or child edits can still cause conflicts.
   - After a squash or rebase landing, conflicts are likely, because the base has the parent's changes as new commits. Where the conflict is in the parent's own lines, the two sides are the same change. Where the child changed a line the parent added, **keep the child's version**. Then check that `git diff --stat origin/<base>...HEAD` shows only the child's own slice. There's no rebase and no force push; the branch history keeps the old commits, and that's fine.
3. Get each push approved and push it. Check each PR's diff is now only its own slice, then carry the merges up the rest of the stack (B4).
4. **Integration branch:** merge the base into it, and get that push approved.
5. If the merge didn't delete the branch, delete it only after no open PR targets it, and after approval.

## 4. When every slice in the integration branch has landed

The integration branch is never merged through a PR, so nothing retargets for it. Retarget the PRs that target it to the base by hand, after approval, check they're still open, and only then delete it.

## 5. A cutover built from the fork point

Once the slices it carries have landed:

1. Merge the base into it. The copied lower-slice commits now match the base's content, so the diff shrinks to the cutover.
2. Resolve the conflicts its description lists. Make any pending decisions with the user first, and port the base's newer code onto the new APIs.
3. Build and test (ask first if the project requires it), get the push approved, then update the description.

## 6. After the last PR lands

1. **Check that the base contains the reference's changes**, on the reference's own files. Collect paths using `git diff --name-only --no-renames -z <fork> origin/<reference>`, preserve NUL delimiters, and compare those paths between `origin/<base>` and `origin/<reference>` with a path-safe script. Do not expand the path list through unquoted command substitution. Remaining differences should be review fixes, follow-up slices, planned exclusions, and the base's own newer changes.
2. **The reference PR:** report the result and ask the user whether to close it (with a comment listing the slice PRs) or leave it open. Do only what they choose.
3. **Clean up**, within the authorized scope, and only once no open PR targets the branch: remote slice branches and the integration branch, local branches (including `work/` and `parked/`), the slicing worktree (`git worktree remove …`), and its build caches.
4. **Tracker:** after approval, comment with the PR list and move the ticket on.
5. **Final report:** a table of PRs (number, title, merged on), plus anything deferred.
