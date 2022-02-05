---
title: My dev setup
date: 2022-02-05
tags: ['neovim', 'nvim', 'dev']
description: Development workflow, what apps I use the most? What configuration I use?
---

Recently, I switched to MAC from Windows. For almost 1 year and 4 months I was working on windows because of my job. On windows I used to work on C#/.Net and some proprietary framework built and managed by the company.

Since college time, I have been using *nix platform and I love to continue my journey with nix. I believe nix is much better than windows in multiple ways. Don't want to discuss it here!

I want to setup my terminal as I know I will be spending most of time working with files, sshing to different machines. Running different type of commands and bla bla! Here are few of things I would like to share.


#### What terminal emulator I use?
It's [iterm2](https://iterm2.com/). If you haven't heard of it. I would highly recommend to check it out. So far I'm not facing any issue and I'm liking it very much. I have configured [oh-my-zsh](https://ohmyz.sh/) as the default `$SHELL`. Along with it I'm using multiple plugins to make my life easier. For example: zsh-autosuggestions, git etc.

**What theme I use with ZSH?** [Powerlevel10k](https://github.com/romkatv/powerlevel10k)


**Why powerlevel10?** Because it show different information at the same time for example: git status, current timestamp, and command execution elapsed time etc. and I'm super lazy to configure on my own. So I thought why not to use already existing option. ^_^

Screenshot of terminal
<img src="/assets/img/my-dev-workflow/iterm.png" alt="iterm2 terminal emulator" />

It is clean and simple and I have configured color scheme according to my taste. If you like it, there is a link at the end of the post of all my dotfiles.


#### What text editor I use?
I use [Neovim(nvim)](https://neovim.io/) enhanced version of VIM. I have been using NVIM from last 3-4 months and I can asure you it is exact similar to VIM with some enhancement and active support and development. Now question arises how I'm able to manage to work on NVIM? If you're thinking of plugins then yes you're right my friend I'm using bunch of plugins and after configuring my editor it works almost same as modern IDE like VSCode, ItelliJ and other editors. 


Using nvim, means(at least for me) you don't want to switch apps/editor and continue using keyboard. Also configuring VIM helps you to fill your curisouty on how these text editor works. How they're able to show you built-in functions, lint errors etc. For all these reason I use VIM. Before discussing further I want to show you editor. 

<img src="/assets/img/my-dev-workflow/telescope.png" alt="telescope nvim" />
<center>On right pane, Telescope plugin to search/browse files</center>


<img src="/assets/img/my-dev-workflow/lspconfig.png" alt="lspconfig nvim" />
<center>On left pane, Language Server Protocol(lsp) returning built-in functions</center>


**List of plugins I have installed right now in my system**
<img src="/assets/img/my-dev-workflow/plugins.png" alt="plugins list" />
<center>Installed plugin list</center>


### What other apps I use?
- Web Browser
- Joplin
- Postman
- Slack
- Rectangle (window manager)


These are the apps I use on regular basis to get my job done. Mostly, I spend time looking at termianl and figuring why things are broken or not working as expected lol or reading/checking something on internet. It's 4:25AM now can't think of anything, pardon. At last, if you want to look at my dotfiles feel free to check here [https://github.com/amustaque97/dotfiles](https://github.com/amustaque97/dotfiles). 

THANK YOU for reading till here! Have a great day ahead!
