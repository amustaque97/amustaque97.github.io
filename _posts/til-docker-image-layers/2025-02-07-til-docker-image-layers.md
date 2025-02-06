---
title: 'The Impact of Combining Commands in Docker Images'
date: 2025-02-07
tags: [blog, writing, life, tech, dev]
description: Learn how combining `RUN` commands in Dockerfiles can reduce image size, improve build performance, and optimize caching for faster, more efficient Docker builds.
---

Today, I was working on a task at the office where I had to set up a few things in a Docker image. As usual, I preferred to write all my operations in a single \`RUN\` command, using \`&&\` to chain commands together. I’ve always done it because it seems cleaner and more efficient, reducing the number of layers created in the image. For example, I typically write it like this:

```
RUN apt-get update && apt-get install \-y curl && apt-get clean
```

Combining everything into a single command seemed like the best approach. But as I worked, I noticed that my teammates had a different opinion. They preferred to break up each command into its own `RUN` statement, with each operation on a separate line. They argued that this approach was easier to understand, making it clear exactly what was happening at each step. For instance:

```
RUN apt-get update  
RUN apt-get install \-y curl  
RUN apt-get clean
```

### **The Moment I Noticed: Layers and Image Size**

I decided to try out my teammates’ approach and see what impact it had on the image. After switching to separate `RUN` statements for each command, I ran the build pipeline and started paying attention to how the layers were being created. I quickly realized something important: each `RUN` command created a new layer. This meant that the more separate `RUN` commands I used, the more layers were created, which ultimately led to a larger image size.

On the other hand, when I reverted to my usual style of combining all commands into a single `RUN` statement, only **one layer** was created. This got me thinking: how does the number of layers affect performance, and is it better to have fewer layers in the image?

### **Technical Details: Layers and Docker Caching**

In Docker, each `RUN` instruction creates a **new layer**. These layers represent different stages of the image, and each one adds to the overall size of the image. The more layers you have, the larger the image will be. This also affects Docker’s caching mechanism.

When Docker builds an image, it tries to reuse layers that haven’t changed, which speeds up the build process. However, if you use multiple `RUN` commands, Docker will have to re-execute every layer after a change, even if the subsequent layers didn’t actually need to be updated.

For example, with multiple `RUN` commands like this:

```
RUN apt-get update  
RUN apt-get install \-y curl  
RUN apt-get clean
```

That’s **three separate layers**. If any of these commands change in the future (say, the package list is updated), Docker will need to re-run all three layers, even if only one of them changed.

But when you combine everything into one command:

```
RUN apt-get update && apt-get install \-y curl && apt-get clean
```

This results in **just one layer**, which reduces the number of layers Docker has to manage and can make builds faster.

### **What I Learned and How It Affects Dockerfiles**

* **Fewer Layers \= Smaller Images**: When you combine `RUN` commands, you create fewer layers, which reduces the overall size of the image.

* **Improved Build Performance**: Fewer layers can speed up the build process since Docker doesn’t have to re-run every step when caching layers. It can use cached layers more effectively.

* **Cache Management**: If you make changes to a Dockerfile, combining commands into a single `RUN` statement means that only the relevant layer will be rebuilt. This can help avoid unnecessary rebuilds of layers that didn’t change.

### **The Better Approach: Combine Commands**

While I still like to use a single `RUN` command for efficiency, my teammates made a valid point about readability. However, from an optimization standpoint, combining related commands into one `RUN` statement is often the better approach. It’s more efficient, reduces the number of layers, and can ultimately make your image smaller and your builds faster.

Here’s how I’ll write it going forward:

```
RUN apt-get update && apt-get install \-y curl && apt-get clean && rm \-rf /var/lib/apt/lists/\*
```

This ensures I’m cleaning up the package manager cache and reducing the final image size.

### **Conclusion**

Today, I learned an important lesson about Docker image optimization. While breaking up `RUN` commands into separate lines can be useful for readability, combining them into a single command often leads to more efficient, smaller, and faster Docker images. By reducing the number of layers and taking advantage of Docker’s caching mechanism, you can streamline your builds.

Next time you’re writing a Dockerfile, try combining `RUN` commands for a more optimized image. It can make a big difference\!

