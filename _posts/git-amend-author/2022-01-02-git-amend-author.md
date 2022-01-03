---
title: How to change your git commit author?
date: 2022-01-02
tags: [git, cli, commit]
description: How to change git commit author?
---

Recently, I started configuring my git details locally based on the project. This helps me in using multiple git profile.
But there could be some instances where I forgot to configure my git then it will show some local machine email address as shown in the image.

<img src="/assets/img/git-amend-author/machine-config.png" alt="Git default config" />

After looking on the internet, I found this simple command to change commit author details
```bash
git commit --amend --author='Full Name <email-address@domain.com>'
```

Here is the live example I used it in one of the project.

<img src="/assets/img/git-amend-author/amend-author.png" alt="Git amend author" />

This is how we can change our git commit author. I find this command really helpful. I don't have to worry even if I'm working on a new device with no configurations at all.
