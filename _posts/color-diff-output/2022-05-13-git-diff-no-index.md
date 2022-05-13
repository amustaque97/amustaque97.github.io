---
title: How to display diff similar to git-diff output for non-git files?
date: 2022-05-13
tags: [git, cli, commit, diff]
description: If files are not part of git working tree how to use git diff command to show diff between files similar to git diff output?
---

These days, I'm spending most of my free time watching [twitch streams](https://www.twitch.tv/anthonywritescode). While watching stream, I learned how to show colour diff between two files, similar to git diff output even if files are not part of current git working tree.

For example, you're working in a non-git directory and want to compare two file differences. There are a bunch of commands you can use, for example `diff` and `vimdiff`. But problem with other tools in my case is I'm already familiar and comfortable with git diff output. I would like to see similar output which is not possible with other tools. That's why I would love to stick with the `git diff` command.

Here is an example of the command:
```console
git diff --no-index <file1> <file2>
```


**Screenshot of above command output:**
<img src="/assets/img/color-diff-output/command.png" alt="git diff --no-index" />




