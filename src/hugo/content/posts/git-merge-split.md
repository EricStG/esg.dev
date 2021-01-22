---
title: "Stupid Git tricks: Merging and splitting repositories"
date: 2020-09-21T14:01:55-04:00
summary: Merging and splitting git repositories without losing history your entire history.
tags:
- git
keywords: git
---

Maybe it's due to working mainly in small shops, but I've had to merge and split git repositories a number of times in the past few years (usually related to folks pushing for microservice architecture, and then changing their minds).

Fortunately, git allows us to do both operations without losing the history.

Needless to say, those operations should not be part of your daily flow, but it's good to have in your toolbox if you ever find yourself in a situation that requires it.

# Warning

Some of the operations described here, especially when it comes to splitting, are extremely destructive as they can rewrite your git history. As a result, other contributors may not be able to push or send pull requests anymore.

Ideally, these operations should be done on fresh clones, and pushed to new remotes.

# Merging

Let's say you have 2 local repositories that you want to merge, called `A` and `B`, and you want to copy the history of `A` into `B`.

## Step 1: Organization

In order to save yourself some headaches, it's better to plan ahead how you want the final repository layout to be. You'll also want to move files around to avoid any conflicts.

For simplicity, I want the end result to be:

```
src/A
src/B
```

In the `A` repository, using whatever tools you want, create the `src/A` directory and move everything in there.

After that, execute `git add -A` to update the index and make sure you `git commit`.

Repeat the process for repository `B`.

And now everything should be where you want it to be.

## Step 2: Pull

From the `B` repository, you will need to `git pull` with the `allow-unrelated-histories` to, surprise, allow the unrelated histories of both repositories to be merged together.

If both repositories are within the same directory, you'd can use this: `git pull --allow-unrelated-histories ../A`.

Once that's done, check the content of your repository and your `git log` to make sure that everything is good, and then you are done!

# Splitting

In this scenario, we'll be reverting the changes we've done above, going from 1 repository containing:

```
src/A
src/B
```

and split them back into 2 repositories, with the content at the root of each ones.

## Danger Will Robinson

Splitting a repository is a highly destructive operation. Not only is it possible to lose the content of your repository, in fact, that's basically how the tool works. It *rewrites the history* and wipes out any traces of the items you have taken out.

Even when things go well, and you can stitch the repositories back into one using the steps above, commits that used to contain changes that were formerly in the same repository will stay separate. That data is gone forever.

## Prerequisite

For this operation, we'll be using a 3rd party tool called [git filter-repo](https://github.com/newren/git-filter-repo/). Note that this tool is a recommendation from the [git filter-branch](https://git-scm.com/docs/git-filter-branch) man pages.

[Install](https://github.com/newren/git-filter-repo/blob/main/INSTALL.md) it for your environment and we'll be good to go.

## Cloning

To makes things easier, do 2 fresh clones of your repository. Each clones will be rewritten with only the content we want.

## Using git filter-repo

Open your command prompt/shell/terminal into the root of your first clone.

Now type `git filter-repo --subdirectory-filter src/A`

This will remove everything that is NOT under `src/A` and move the content to the root of the repository.

Repeat the same operation for your second clone, changing `src/A` to `src/B`, and you are done!

# Conclusion

These are some of the ways I've used in the past to split and merge git repositories. What are yours? Hit me up on [twitter](https://twitter.com/Eric_STG)!
