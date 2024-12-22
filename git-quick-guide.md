# Git Commands
## Outline
- [Apply Patches](#apply-patches)
- [Branch Prunning](#branch-prunning)
- [Repo Reconstruction](#repo-reconstruction)
- [Branch Update](#branch-update)
- [Fix Branch](#fix-branch)
- [Count Commits](#count-commits)
- [Git Config](#git-config)
- [Git Archives](#git-archives)
- [Backlog](#backlog)

## Apply Patches
### From another branch into current
```sh
git checkout '<A>' # if you are not there already
git format-patch '<C>~1'..'<B>' --stdout | git apply -
```
Apply changes between `C~1` and `B` to the working tree of `A`. Note that we _do not_ want to undo `A~1` so we cannot simply apply `B..A`.
```
  * A
  |
  *
  |
  | * B
  | |
C * |
  |/
  * 
  |
```

## Branch Prunning
Prune local branches that have already been merged upstream. Then remove all local branches that have been merged to main. Optionally, you may remove non-merged branches that do not exist in origin.
```sh
# remove branches from origin
git fetch --prune
git checkout main # IMPORTANT
# see below for explanation
git branch --merged origin/main | grep -v '^\*' | xargs -n 1 git branch -d
# git remote prune origin # optional
```
The first comand in the pipeline marks with `*` the branches have _not_ been merged into your current one (the one that is checked out). **Hence why it is important that you checkout main**. Then, `grep` filters out the ones with asterisk. Finally, `xargs` passes at most one argument from each line in `stdin` to `git branch -d`.

## Repo Reconstruction
### Goal
Recreate a repo from its root up to one commit in its history. From that point onwards generate patches for all commits (of the original repo), but filter the diffs such that _only_ changes for an arbitrary set of files is included.

The commit order and metadata should be preserved. To make things easier make two new directories for the two sets of patches.

### Commands
```sh
cd '<old_repo_path>'
mkdir '<dir1>' '<dir2>'
# generate unconditional patches
git format-patch -'<n>' '<ref>' -o '<dir1>'
# generate filtered patches
git format-patch '<ref>'..HEAD -o '<dir2>' -- '<file_1>' '<file_2>' ... '<file_k>'
mkdir '<new_repo_path>'
cd '<new_repo_path>'
git init
# apply patches in new repo
git am '<dir1>'
git am '<dir2>'
```
- `<ref>`: most recent commit you want to fully preserve along with all its ancestors
- `<n>`: number of ancestors from `<ref>` to root.

> NOTE: globbing in Unix shells order lexicographically.


Optionally, you may inspect the filtered patches with this command before you apply them
```sh
git format-patch '<ref>'..HEAD --stdout -- '<file_1>' '<file_2>' ... '<file_k>' | less
```

### Example
Consider the following commit history where `~/repo_old` and `~/repo_new` directories already exist.
```
(latest commit)
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
* C8 <- preserve this and all its parents
|
*
|
*
(root)
```

```sh
cd ~/repo_old
mkdir ../dir1 ../dir2
# generate unconditional patches
git format-patch -3 C8 -o ../dir1
# generate filtered patches
git format-patch C8..HEAD -o ../dir2 -- X Y
cd ~/repo_new
git init
# apply patches in new repo
git am '<dir1>'
git am '<dir2>'
```
Optionally, clean up the directories with the patches.

## Branch Update
In the two strategies below always keep in mind that when updating a branch from `main`, `SOURCE` is `main`. `TARGET` is always the branch getting new commits or commit.
### By Merging
```sh
# update target and/or source (if needed)
git checkout '<ref>'
git pull origin '<ref>'
# start merge
git checkout SOURCE
git merge TARGET
```
### By Rebasing
```sh
# update target and/or source (if needed)
git checkout '<ref>'
git pull origin '<ref>'
# start rebase
git checkout SOURCE
git rebase TARGET
```
if already present upstream
```sh
git push --force-with-lease
```

## Fix Branch
### Mistakely Add Commits to Merged Branch
Branch `A` was merged to main. `A` has `n` commits mistakenly added  to it. They were meant to go in another branch whose base is `main` (the merge point of `A`). There is also a tag at the `HEAD` of `A`. Assumption is made that nothing has been pushed to origin.
```
A * tag-1
  |
 ...
  |
  *
  |
  | * main
  |/
  *
```
Run the following
```sh
git checkout A
git branch B
# make sure your git status is clean
git reset --hard HEAD~'<n>'
# checkout main and pull (if needed)
git checkout B
git rebase main
git tag -f tag-1
git checkout main
git merge --ff-only B
git branch -d B
```
results in
```
  * main, tag-1
  |
 ...
  |
  *
  |
  *
  |
A *
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

### Count Filtered by Specific Files Changed
```sh
git rev-list --count A..B -- file1 file2 ... filen
```

## Git Config
### Config Default Branch Name
Simply run
```sh
git config --global init.defaultBranch main
```
if you want your new name to be main.

## Git Archives
### Zip Archive of Commited Changes only
Create a zip out of the repo that includes only committed changes **without the `.git` directory**, and put it one folder up from current directory.
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

### Push tags upstream
[stackoverflow answer](https://stackoverflow.com/a/5195913). Essentially use
```bash
git push origin tag '<tag_name>'
```
