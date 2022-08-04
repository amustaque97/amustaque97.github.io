---
title: How to search text pattern in git history? 
date: 2022-08-05
tags: [git, cli, log]
description: Want to know at what time text got added or remoted in your entire git history? 
---

Assume you're contributing to a project from past one year with multiple contributors or team members. The project source uses Git as version control system. I'm sure if you're reading this, then you must be aware of Git. Git is a powerful tool. It contain countless options like logs, diff, describes and so on. Since last year, I have been contributing to multiple open source projects. Based on the situation, sometimes I'm interested in knowing when we added or removed some library or method, what was the first commit and what is the recent commit?


Today, I learned a command on how to search a pattern or text in the git commit history till now.

```console
git log -G <pattern> -- <file_name>
```

How does this command help? Let's read the man page and understand what does it say?

```console
-G<regex>
 Look for differences whose patch text contains added/removed lines that match <regex>.

 To illustrate the difference between -S<regex> --pickaxe-regex and -G<regex>, consider a commit with the following diff in the same file:

     +    return frotz(nitfol, two->ptr, 1, 0);
     ...
     -    hit = frotz(nitfol, mf2.ptr, 1, 0);

 While git log -G"frotz\(nitfol" will show this commit, git log -S"frotz\(nitfol" --pickaxe-regex will not (because the number of occurrences of that string did not
 change).

 Unless --text is supplied patches of binary files without a textconv filter will be ignored.

 See the pickaxe entry in gitdiffcore(7) for more information.
```

In my case, I use this command to search for any method or library keyword to check when was it first introduced in the project? OR When it got removed from the project?

Commands like this help us in saving a lot of time if we do the same work manually. 

