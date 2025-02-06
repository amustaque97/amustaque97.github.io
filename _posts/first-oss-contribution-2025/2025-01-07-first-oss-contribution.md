---
title: 'My 2025 Open-Source Debut: Adding a Path Opener to Ghostty!'
date: 2025-01-07
tags: [blog, writing, life, tech, dev]
description: Learn about my first open-source contribution of 2025 where I added a path opener feature to Ghostty, a terminal emulator, and how it improved my daily workflow and helped the community.
---

2025 started off with a bang for me, as I made my first open-source contribution of the year! It’s a moment I’ve been eagerly waiting for, and it has left me feeling a mixture of excitement, accomplishment, and pride. It all began with a spark of curiosity and a bit of exploration into a new programming language, Zig, which led me to a fascinating project by one of HashiCorp's co-founders: **Ghostty**, a terminal emulator.

## How It All Began

A couple of months ago, I was trying out Zig for fun. Just a bit of curiosity to see what the language had to offer. While exploring new tech, I stumbled across **Ghostty**, a terminal emulator being built by a passionate developer who had previously co-founded HashiCorp. Ghostty immediately caught my attention for its innovative approach, and before I knew it, I found myself in the project's Discord community, offering feedback and engaging in discussions.

After some time interacting with the team, I was invited to the code repository. This was my first real taste of contributing to an open-source project, and it felt amazing! I started using **Ghostty** as my daily driver, appreciating its unique features and performance. It was smooth, fast, and had great potential, but there was one feature I wished it had.

## The Feature Request That Sparked My Contribution

As I was using Ghostty day-to-day, I shared it with my colleagues at the office. They were excited to try it out too, but one of them mentioned a feature they missed from other terminal emulators like iTerm2: **the ability to hover over a file path with the mouse and open the file by pressing `Cmd + Click`.**

It seemed like a simple feature, but it was something that could drastically improve the usability of Ghostty. I immediately reached out to the team on Discord with this suggestion, and much to my excitement, they loved the idea. We decided to clone the repository and give it a shot. The challenge? I had never worked on system software before, so the learning curve was steep.

## Diving into System Software: A Learning Experience

I spent 2-3 days getting up to speed with the code. I’ll admit, it was a bit intimidating at first. I was working with concepts I had never dealt with before, like **Metal rendering** and **OpenGL**, and I had no experience with them. But the more I dived into the code, the more intrigued I became. It was like solving a puzzle, with each piece of knowledge slowly clicking into place.

While navigating the codebase, I discovered that **Ghostty** uses **regex** to highlight text as hyperlinks. This was my breakthrough moment. I knew I could use this mechanism to highlight file paths and add the "hover-to-open" functionality.

## Adding the Feature: Writing the Code

With my understanding of the code growing, I wrote the regex to identify file paths and added the necessary code to allow for opening files when hovering and clicking on them. It wasn’t easy, but it was definitely rewarding. I recorded a demo video showing how the feature worked, feeling a wave of excitement as I tested it out for the first time.

Once I was happy with my changes, I raised a **Merge Request (MR)** for review. I was nervous but hopeful. The best part? The review process went smoothly! I was thrilled when the MR was reviewed and **merged the same day**. 

You can check out the **Merge Request** here: [PR #4713 on Ghostty](https://github.com/ghostty-org/ghostty/pull/4713).

## The Moment of Accomplishment

Getting my first contribution merged felt like such an incredible achievement. It was a perfect blend of curiosity, hard work, and collaboration. Not only had I helped improve a project I believed in, but I also added a feature that my colleagues and I were genuinely excited about.

One of the most fulfilling parts of this experience was how the **new release of Ghostty**, which included the improved version of the path opener, was exactly what my friend had asked for. It was truly amazing to see my contribution go from an idea shared on Discord to a fully functional feature that improved the daily workflow for many people.

## Reflecting on the Journey

Looking back on this experience, I can’t help but feel a deep sense of happiness and accomplishment. Not only was I able to contribute to a project that excites me, but I also learned a lot along the way. Working with system software, understanding rendering technologies like Metal and OpenGL, and using regex in ways I had never imagined were all valuable lessons that expanded my knowledge.

Most importantly, this experience reaffirmed why I love open-source. It’s a space where collaboration and passion intersect, and anyone, no matter their experience level, can contribute something meaningful. It also reminded me of the power of community—the feedback and support I received from both the project’s maintainers and my colleagues made this whole experience unforgettable.

## Conclusion: A New Year, A New Beginning

This first open-source contribution of 2025 has set the tone for the rest of the year. It’s given me a renewed sense of motivation to continue contributing to open-source projects, to learn new things, and to help others in any way I can. It feels like the beginning of a new chapter, and I can’t wait to see what the future holds.

For anyone looking to dive into open-source, I highly recommend it. Even if you're new to the field, like I was with system software, you’ll find that the experience is incredibly rewarding. It’s a journey of learning, growing, and contributing to something larger than yourself.

Here’s to more contributions, more learning, and more amazing open-source projects in 2025!