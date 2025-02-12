+++
date = '2025-02-12'
draft = false
title = 'git rebase for people that dont really care'
+++

# Boring intro
This article is for people who use `git` but did not go the extra mile to learn
its features. There are no boring technical details and I might cover only a small
fraction of `git rebase`. I made this article to be practical and "down-to-earth" guide.

## Why rebase?
Rebasing your git repository would be useful in these scenarios:
+ You want to change the commit messages
+ You want to fix already commited changes
+ You want to squash (connect together) small commits scattered around `git log`
+ You want to delete the commit and its changes

Keep in mind that git repository is like a literal, real-world tree -- if you
start to cut or twist the older branches, younger branches will become unstable
or just fall off. Try to be mindful about repository commit history and how
textfile edits can be stored against other changes -- for example, you cant
expect `git` to do a conflict-less rebase when you make the commit changing a file `hello.c`
be earlier than the commit with its initial appearance (creation).

After a rebase you have to use `git push --force` if you changed the commits
that were already pushed, so it is time to stop sporadically pushing and start
thinking about making repository history to be readable and easily manageable👽👽👽.

# The fun stuff
## Quick start
To quickly start the rebase, run `git rebase -i HEAD~5` (`-i` would mean
`--interactive`). `HEAD~5` will index a list of 5 recent commits. If you want
to avoid any potentional conflicts, use the least needed amount of commits
relative to `HEAD`, unless you are trying to manage old commits, of course.

`git rebase -i HEAD~5` will open a text editor window that may look like this:
```
upick 16dcd7f updated README.md
pick 8f12ed8 added pandoc rpm for installation
pick 6b74a7a added reference tasklist for GNOME extension management
pick 1b7f65e added GNOME extension management part and its unused prototype
pick c57b3f3 gnome shortcut for ptyxis will now use `--new-window`

# Rebase 9b93623..c57b3f3 onto 9b93623 (5 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup [-C | -c] <commit> = like "squash" but keep only the previous
#                    commit's log message, unless -C is used, in which case
#                    keep only this commit's message; -c is same as -C but
#                    opens the editor
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
#         create a merge commit using the original merge commit's
#         message (or the oneline, if no original merge commit was
#         specified); use -c <commit> to reword the commit message
# u, update-ref <ref> = track a placeholder for the <ref> to be updated
#                       to this position in the new commits. The <ref> is
#                       updated at the end of the rebase
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
```

The bottom part is really verbose and self-explanatory. Do not delete any lines
in this file -- `git` will just delete the commit and its changes.

### Changing commit messages
Go to a line with a commit where you want to change the message and replace the
first word with `reword`. Examine the opened textfile syntax and make the necessary changes.

### Managing commited changes
#### Editing commits directly
To edit a commit, change the `pick` word to `edit`. Make the necessary edits.
You can inspect your changes as when adding changes normally -- `git status`
and `git diff` will work as expected. When you are finished, `git stage` (or
`git add`👽) your changes, run `git commit --amend` and `git rebase --continue` after that.

#### Squashing commits into one
If you are making small and focused commits when editing your stuff -- you are
doing it right. However, sometimes small and focused could be too small and too
much 😃. Move the more recent commits (in their respective order) under the
initial commit (for example, for a new function) and change the first word to
`squash`. If you pick and arrange your commits right, there will be no need to
manually resolve conflicts and you will be offered to change the commit
message. Now your 20 commits of your futile attempts and small changes can be
presented as one commit with a presentable message.

### Deleting commits and their changes
In interactive text buffer, just delete the line or change the first word to
`drop`. If you do not do anything stupid (like messing up the order) -- it will not ask you for manual
conflict resolution.
