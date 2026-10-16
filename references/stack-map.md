# Stack map template

Fill this in when you plan the split (phase A), and keep it up to date as branches move. Keep it in your notes or scratchpad. Don't commit it to the repo unless the team wants it there.

```markdown
Ticket / epic: <ID>
Host: <GitHub | GitLab | Bitbucket>, merges by <merge commit | squash | rebase>, retargets on merge-and-delete: <yes | no | unchecked>
Base: <main, master, develop, or the project's integration branch>
Reference: <branch> (#<n>, open, unchanged)
Fork point: <sha>     Pinned base for cutting: <sha>
Integration branch: <name or "none"> = base + PRs <…>

| # | Branch | PR | Targets | Start point | Other PRs target it? | Layer / owns |
|---|--------|----|---------|-------------|----------------------|--------------|
| 1 | feature/<ID>-01-<slug> | <link> | develop | develop | via stack-base | modules, no callers |
| 2 | feature/<ID>-02-<slug> | <link> | develop | develop | via stack-base | additive API |
| 3 | feature/<ID>-03-<slug> | <link> | feature/<ID>-stack-base | stack-base | **yes: PR 4** | compiled but unused |
| 4 | feature/<ID>-04-<slug> | <link> | feature/<ID>-03-<slug> | PR 3 | no | cutover |

Working worktree: <path>
```

The "Other PRs target it?" column lists the PRs to check right after that branch merges (rule 10): are they still open, and retargeted?

## Finding the stack when nobody wrote it down

| Host | Command |
|---|---|
| GitHub | `gh pr list --state all --search "<ID>" --json number,title,state,headRefName,baseRefName` |
| GitLab | `glab mr list --all --search "<ID>"`, or the MRs API (`source_branch`, `target_branch`, `state`) |
| Bitbucket | The REST API or the team's MCP tool (`source.branch.name`, `destination.branch.name`, `state`) |

The `[k/N]` in titles, or the branch numbers (`-01-`, `-02-`, …), give the order.

## Checking branches against their targets

```bash
$SKILL/scripts/stack-status.sh feature/<ID>-01-<slug>:develop feature/<ID>-04-<slug>:feature/<ID>-03-<slug>
```

## Finding which slice owns some lines

Start from the branch the comment is on, with the line numbers as that PR shows them. List every slice with its target, lowest first, and run this in bash:

```bash
B=origin/<branch the comment is on>; FILE=path/to/file; LINES=120,160
PAIRS="feature/<ID>-01-<slug>:develop feature/<ID>-04-<slug>:feature/<ID>-stack-base"   # every slice:target, lowest first
git blame -s -L "$LINES" "$B" -- "$FILE" | awk '{sub(/^\^/, "", $1); print $1}' | sort -u | while read -r c; do
  subj=$(git log -1 --format=%s "$c")
  for p in $PAIRS; do s=${p%%:*}; t=${p#*:}
    if git log --no-merges --format=%s "origin/$t..origin/$s" | grep -qxF -- "$subj"; then echo "$c ($subj) belongs to $s"; break; fi
  done
done
```

`git blame` names the commits that last changed those lines. Each commit is matched by subject against each slice's own commits (target..slice), so a cherry-picked copy, such as a cutover's copies of lower slices, is credited to the slice that made the original. A commit that matches no slice came from the base. Right after a split, `split-coverage.sh -v` also lists each file's owning slice.

In zsh, split command output into an array (`B=($(…))`) before passing a branch list to a script, because zsh doesn't split `$var` into words.
