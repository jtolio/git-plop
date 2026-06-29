# git-plop

`git-plop` is a "Git forge" agnostic stacked-PR workflow tool.

## Basic use

Let's say you're using a branch-based code review Git forge (Gitlab, GitHub, etc). You have a number
of reviews out, some of them depending on others. All of your remote branches have a common prefix.
You want to amend one commit and then update all changed remote branches.

First, set your remote branch prefix. If you skip this step, your local computer's username will
be chosen. (`git config plop.branch-namespace branch-prefix`).

Let's say you have three reviews in your Git forge:

 * change-1: Makes a change, consists of two commits, intended to merge to main.
 * change-2: Makes a change dependent on change 1, consists of two commits. During review, targeted
    at change-1, not main.
 * change-3: Makes an independent change, targeted at main.
 
Maybe `git log --decorate --graph --oneline` might show:

```
* 35850d2 (origin/branch-prefix/change-3) Make an independent change
| * b950bf1 (origin/branch-prefix/change-2) Make a dependent change
| * 431a445 Make a dependent change prep
| * cf24fc6 (origin/branch-prefix/change-1) Make a change
| * 185a3e9 Make a change prep
|/  
* 8614f03 parent
```

You want to make a change to a commit that is part of your `change-1` branch and you want to have
all dependent changes updated. Let's use `git-plop`!

Take a look at your remote branch tips. Note that without the `-a` option, this will not show 
remote branches that are ancestors of other remote branches:

```
$ git plop remote -f
b950bf1328 fc6mfqs7jp5vdigwhiug -> branch-prefix/change-2   Make a dependent change (4)
35850d2991 jkj3fquspzk3oaglrdnx -> branch-prefix/change-3   Make an independent change (1)
```

The first column is the Git hash of that commit tip. The second column is the git-plop change id.
The next column is the name of the remote branch git-plop will push that change id to. Finally,
the count is the number of unmerged commits that commit branch includes.

You can choose the commit you want to do development on:

```
$ git reset --hard b950bf1328
```

Let's take a look at your current stack:

```
$ git plop log
b950bf1328 fc6mfqs7jp5vdigwhiug -> branch-prefix/change-2   Make a dependent change
431a44523b br4ys4jwdknxk44qlrm4 -> _                        Make a dependent change prep
cf24fc68ae 6lt5cbjfguiotpthbm2x -> branch-prefix/change-1   Make a change
185a3e953a ele2hf23e5wbkhnud2it -> _                        Make a change prep
```

Note the change ids and the Git refs.

Then, do your edits. You can run `git rebase -i` like normal, or you can use `git plop rebase` to
use the right interactive rebase merge-base.

Now let's take a look at your current stack again:

```
$ git plop log
64cb312338 fc6mfqs7jp5vdigwhiug -> branch-prefix/change-2   Make a dependent change
efc78bc5ee br4ys4jwdknxk44qlrm4 -> _                        Make a dependent change prep
c94e49fe57 6lt5cbjfguiotpthbm2x -> branch-prefix/change-1   Make a change
a1ed9709d4 ele2hf23e5wbkhnud2it -> _                        Make a change prep
```

Note that your git refs changed but your change ids did not!

Now, you can update both the `change-1` and `change-2` branches by running:

```
git plop lob
```

This will update all branches in the stack.

## Full usage

```
git-plop

git-plop enables updating multiple remote branches from unmerged commits on a
single local branch.

git-plop works primarily using a pseudo "change id", akin to a Gerrit or
Jujutsu change id, calculated as a hash of the author's name, email, time, and
timezone. Author information within a commit rarely changes across rebases or
cherry-picks, and authors rarely commit more than once a second. Note that
git-plop will behave strangely if a particular author commits more than once a
second.

git-plop keeps a small database in the current repo's .git/info/plop/
(overridable with git config plop.db-path) which includes mappings of change
ids to target branches.

git-plop aim [ref [ref [ref ...]]]
git-plop aim --all [-r remote] [ref]
git-plop aim [ref] -t target-branch

    This command sets the target branch of a particular ref in the plop
    database.

    If no arguments are provided, it generates a target branch name and sets it
    for HEAD if HEAD does not already have one.
    If --all is provided, then all unmerged commits are aimed with a synthetic
    branch if they don't already have one. See 'git-plop unmerged'.
    If one or more arguments are provided without -t, each argument is set to a
    new synthetic target branch if one does not exist.
    If -t is provided, then the given ref (or HEAD, if none is given) is set to
    the given branch.

    If target-branch is not specified, it is generated with the following
    format:

        <namespace>/<trimmed-message>-<shortcode>

    <namespace> is determined by either looking in git config for
    'plop.branch-namespace' or using the current system username.
    <trimmed-message> is determined by taking the first 12 alphanumeric
    characters of the commit message. Non-alphanumeric characters are replaced
    with dashes.
    <shortcode> is 6 random base 32 digits from /dev/urandom.

git-plop disarm [ref [ref [ref ...]]]

    This command undoes git-plop aim for the given refs, defaulting to HEAD if
    none are given.

git-plop id [ref]

    This command shows the git-plop change id for the ref, defaulting to HEAD.

git-plop lookup [ref]

    This command shows the git-plop destination branch for the ref, defaulting
    to HEAD.

git-plop unmerged [-r remote] [ref]

    This command shows all commits on the current branch that are currently
    unmerged to the remote branch. It does this by looking at the ref, finding
    the merge-base of ref and the current remote tracking branch (or the
    default branch on the remote if no tracking branch is found), and then
    looking at that chain.
    The commit chain from merge-base to ref must be a unique linear path. If
    multiple paths exist, an error is raised.

    If remote is not specified, it defaults to the default remote that git push
    would use.
    If ref is not specified, it defaults to HEAD.

git-plop lob [-r remote] [ref]

    For each commit in the unmerged set, this command considers whether it has
    been aimed. If so, it adds it to a refspec to push to the remote with
    --atomic and --force-with-lease. This will update the specified remote
    branch for every commit that is found to have a destination. No other refs
    besides aimed refs are pushed.

git-plop log [-r remote] [ref]

    This command shows the current unmerged commits along with their change ids
    and destination branches.

git-plop rebase [-r remote] [ref]

    This command starts an interactive rebase of all of your changes in the
    current stack.

git-plop remote [remote] [-a] [-f]

    This command looks in the appropriate remote for all remote branches in
    your configured namespace (see 'git-plop aim' for namespace documentation).
    It then logs the tips of each.
    If [-a] is not provided, only branches that are not ancestors of other
    branches will be shown.
    If [-f] is provided, `git-plop fetch -q` runs first.

git-plop branch [-r remote] [ref]

    For better usability with other tools, this tool will make or update local
    git-plop branches in a 'plop/' branch namespace for every aimed git commit
    in the local database.

git-plop fetch [remote] [-q]

    This command runs `git fetch` against the appropriate remote, then looks in
    that remote for all remote branches in your configured namespace (see
    'git-plop aim' for namespace documentation). Once it identifies remote
    branches in your namespace, it makes sure that those changes have all been
    aimed to those remote branches.
    If [-q] is provided, no update output is shown.

git-plop unbranch

    Removes branches from 'git-plop branch'

git-plop clean

    Removes all git-plop data.
```

## License

MIT
