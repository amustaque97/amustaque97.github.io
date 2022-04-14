---
title: How to checkout pull request locally?
date: 2022-04-14
tags: [git, cli, commit, pull-request]
description: How to checkout pull request in local?
---

It has been a few months that I have joined **Gutenberg** project 🎉  on the Github platform. Now I can do PR(pull request) review and add a comment if required, or if everything looks good, I can merge it. I started with the PR review. I found that I don't know how to checkout to PR changes 😔! 

I checked online and found the easiest way to checkout PR is to use *Github CLI tool*. On Github, If you navigate to the PR. You will see something like this.

<img src="/assets/img/checkout-pull-request-locally/checkout-pr-gh.png" alt="Github CLI PR checkout" width="50%" height="50%" />

Command mentioned in the screenshot solves the problem; however, I was curious how can I do the same without the Github CLI tool. After going through the documentation, I found a command as below.
```console
git fetch pull/<PR-ID>/head
```

After running the above command as you can see that a new branch is fetched (as shown in the below image).
<img src="/assets/img/checkout-pull-request-locally/git-pull-checkout.png" alt="Git fetch pull" width="100%" height="100%" />


Now I did `git checkout FETCH_HEAD` and checked out the same PR. 
