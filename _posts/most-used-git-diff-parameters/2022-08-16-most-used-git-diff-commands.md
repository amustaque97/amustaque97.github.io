---
title: Most used git diff commands in my workflow
date: 2022-08-16
tags: [git, cli, diff]
description: Git diff commands that I use mostly in my entire dev workflow 
---

In my daily job, I use Git for almost everything related to version control systems. And If you're familiar with Git. I assume you might have heard/used the `git diff` command. `git diff` command show all changes we made. I use the same command `git diff` to understand changes additional parameters that show or hide some extra information based on the use case.
In this post, I write about the two parameters/flags I use most of the time while executing the `git diff` command.

Here are the two-parameter names that I use.
1. --staged
2. --stat

Lets us first talk about the first parameter `--staged`. I use this parameter when I have added my changes in the stage area; in other words I run this command after running command `git add`  to see the changes. Here is the man-page documentation.

```console
git diff [<options>] --cached [--merge-base] [<commit>] [--] [<path>...]

This form is to view the changes you staged for the next commit relative to the named <commit>. Typically you would want a comparison with the latest commit, so if you
do not give <commit>, it defaults to HEAD. If HEAD does not exist (e.g. unborn branches) and <commit> is not given, it shows all staged changes. --staged is a synonym
of --cached
```

Oh! I never knew before that `--staged` is a synonym of `--cached`. Good to learn that.

Second command I would like to share is `--stat`. Let's learn this parameter with the help of an example. Let's say you are working on a project and have made a few changes to several files. To see the changes you've made, you can simply run ‘git diff' or if you want to see all the files that you've changed, you can run ‘git status'. What if you want to understand the number of lines that you added or deleted in which file?So the 'git diff --stat' command will tell you that. Now you can find out more about your changes, such as major or important file changes, anything like that. Here's the screen capture of how the output looks.

<img src="/assets/img/most-used-git-diff-parameters/git-diff-stat.png" alt="Git diff stat command output" width="100%" height="100%" />


That's all I wanted to share, keeping in mind that we can use a combination of those two or more at the same time. Thank you so much for reading till the end.

