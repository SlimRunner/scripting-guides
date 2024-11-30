# Git Commands
## Outline
- [apply patches](#apply-patches)
- [brach prunning](#branch-prunning)
- [patch reconstruction](#selective-reconstruction-by-patches)
- [update branch](#updating-branch)
- [count commits](#count-commits)
- [config default branch name](#config-default-branch-name)
- [zip git archive](#zip-git-archive)
- [backlog](#backlog)

## Apply Patches
### From another branch into current
```sh
git format-patch '<ref>'..'<ref>' --stdout | git apply -
```
I used this to apply changes from branch `A` to branch `B` where the commits look like follows:
```
*   A
|
*
| * B
* | main
|/
* 
|
```
However, I only want to apply to `A`'s working tree the changes that `B` would have applied to `main~1`, and without undoing `A~1` either.

## Branch Prunning
Prune local branches that have already been merged upstream. First we remove the branches from origin
```sh
git fetch --prune
```
> BEFORE you do the following **checkout main**
This deletes all local branches which have been merged upstream.
```sh
git branch --merged origin/main | grep -v '^\*' | xargs -n 1 git branch -d
```
The first comand in the pipeline marks with `*` the branches have _not_ been merged into your current one (the one that is checked out). **Hence why it is important that you checkout main**. Then, `grep` filters out the ones with asterisk. Finally, `xargs` passes at most one argument from each line in `stdin` to `git branch -d`.

Optionally to remove **non-merged** branches that no longer exist in upstream
```sh
git remote prune origin
```

## Selective Reconstruction By Patches
### Goal
Recreate a repo from one point in its history, and from there on apply a selected subset of patches (from the original repo) which contain _only_ diffs for a hand picked set of files.

The commit order and original message should be preserved.

### Commands
```sh
git format-patch -'<n>' '<ref>' -o /path/to/dir1
```
Here `<ref>` refers to the most recent commit whose ancestors you want to preserve (including the ref itself), and `<n>` is the number of ancestors plus 1. Ideally, use git's automatic numbering so that applying the patches becomes easier (since original order must be preserved).
> NOTE: globbing in Unix shells order lexicographically.

```sh
git format-patch '<ref>'..HEAD -o /path/to/dir2 -- '<file1>' '<file2>' ... '<fileN>'
```
`<ref>` is the same as before, i.e. it refers to  the most recent commit whose ancestors you want to preserve. Then just list the files you want to filter by.

You can inspect the contents of the patches without writing to files with
```sh
git format-patch '<ref>'..HEAD --stdout -- '<file1>' '<file2>' ... '<fileN>' | less
```

Finally, in order to reconstruct:
```sh
git am /path/to/dir1/*.patch
git am /path/to/dir2/*.patch
```

### Example
Consider the following commit history
```
* C1 <- diff files: Y, Z, W, V
|
* C2 <- diff files: X, Y
|
* C3 <- diff files: Z
|
* C4 <- diff files: V, W
|
* C5 <- diff files: X, Y, Z, V
|
* C6 <- diff files: X, V
|
* C7 <- diff files: X
|
* H <- preserve this and all its parents
|
*
|
*
```

We want to create a repo that contains all the commits from the oldest up to `H`. Then we want to apply subsequent commits _only if_ they changed file `X` or file `Y`. Further, in the case of `C5` the patch should exclude the diffs for `Z` and `V`.

In this example we would run in the current repo
```sh
git format-patch -3 H -o /path/to/dir1
git format-patch H..HEAD -o /path/to/dir2 -- X Y
```

Then in the new repo
```sh
git am /path/to/dir1/*.patch
git am /path/to/dir2/*.patch
```

## Updating Branch
### By Merging
```sh
# update target branch (if needed)
git checkout SOURCE
git pull origin SOURCE
# start merge
git checkout TARGET
git merge SOURCE
```
### By Rebasing
```sh
# update base branch (if needed)
git checkout SOURCE
git pull origin SOURCE
# start rebase
git checkout TARGET
git rebase SOURCE
```
if already present upstream
```sh
git push --force-with-lease
```

## Count Commits
Count commits in between `A` (exclusive) and `B` (inclusive)
```sh
git rev-list --count A..B
```
If `A` is `B` itself or `B`'s child, then the result will always be 0. For a concrete example, the following should result in the number `n`
```sh
git rev-list --count HEAD~n..HEAD
```

The above is true **ONLY IF** the history between the refs is linear. For non linear histories git will count _all nodes_ which connect `A` and `B` in the same fashion of exluding `A` and including `B`.

reference: https://stackoverflow.com/a/31998123

## Config Default Branch Name
Simply run
```sh
git config --global init.defaultBranch main
```
if you want your new name to be main.

## Zip Git Archive
Create a zip out of the repo that includes only committed changes along the `.git` directory, and put it one folder up from current directory.
```sh
git archive --format=zip --output=../'<name>'.zip '<ref>'
```

> NOTE: the `--flag=value` syntax is optional. `--flag value` works as well.

reference: https://git-scm.com/docs/git-archive

## Backlog

### Stash only unstaged files
[stackoverflow question](https://stackoverflow.com/q/20028507)

- [A1](https://stackoverflow.com/a/20028585)
  - good if your goal is to definitely commit after you've tested
- [A2](https://stackoverflow.com/a/34681302)
  - decent if your goal is NOT to commit after you've tested (i.e. you need to bring your other changes before you commit)
  - pro1 no need for clean up (temp files)
  - con1 is that it creates a temporary commit
  - con2 you might be unable to commit (commit hooks)
- [A3](https://stackoverflow.com/a/24899847)
  - does not use stash at all it simply creates a diff which you can remove and re-apply. After you're done you delete the diff
  - pro1 very simple
  - con1 creates a temporary file you must store somewhere
  - con2 you have to delete the file after you're done
  - note `-R` is short for `--reverse`
