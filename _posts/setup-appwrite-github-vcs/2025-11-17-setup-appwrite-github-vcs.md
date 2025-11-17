---
title: Setting Up VCS Integration in Appwrite
date: 2025-11-17
tags: [appwrite, github, integration, dev ]
description: I’ll walk you through how to set up GitHub VCS integration for a local Appwrite instance — based on my own notes while configuring it.
---


If you’ve ever wished your **local Appwrite environment** could automatically deploy your functions or web code on every `git push`, this guide is for you. Setting up Version Control System (VCS) integration might sound intimidating, but once you walk through it step-by-step, it feels surprisingly straightforward.

In this short post, I’ll walk you through how to set up GitHub VCS integration for a local Appwrite instance — based on my own notes while configuring it.



## 🧰 What You Need Before Starting

- A local Appwrite instance running via Docker Compose  
- A GitHub account  
- Basic understanding of Git

That’s it.



## 1. Create a GitHub App

To let Appwrite talk to GitHub, you first need to create a *GitHub App*.

Head over to:  
**GitHub → Settings → Developer Settings → GitHub Apps → New GitHub App**

Fill in the basics:

- **Name:** Anything you like, e.g., *Appwrite Local Dev*
- **Homepage URL:** `http://localhost`
- **Callback URL:**  
  `http://localhost/v1/vcs/github/callback`
- (Optional) **Webhook URL:**  
  `http://localhost/v1/vcs/github/events`

For permissions, give:

- **Contents:** Read & Write  
- **Metadata:** Read-only  
- (Optional) Pull Request + Webhook permissions if you want richer automation later  

Create the app, then collect the important credentials:

- App ID  
- Client ID  
- Client Secret  
- Private Key (`.pem` file)

These will be plugged into Appwrite next.



## 2. Add Credentials to Appwrite

Open your Appwrite `.env` file and add:

```bash
_APP_VCS_GITHUB_APP_NAME=your-app-name
_APP_VCS_GITHUB_APP_ID=123456
_APP_VCS_GITHUB_CLIENT_ID=Iv1.xxxxx
_APP_VCS_GITHUB_CLIENT_SECRET=your-secret
_APP_VCS_GITHUB_WEBHOOK_SECRET=your-webhook-secret
```

For the private key, the easiest way is to base64 encode it:

```bash
cat myapp.private-key.pem | base64 | tr -d '\n'
```

Then put that into:

```bash
_APP_VCS_GITHUB_PRIVATE_KEY="base64-encoded-key"
```

Restart Appwrite:

```bash
docker-compose down
docker-compose up -d
```



## 3. Install the GitHub App

Visit your GitHub App page → **Install App**  
Choose your account and select the repo(s) you want to use.



## 4. Connect a Repository in Appwrite

Open your local Appwrite console → navigate to the project → **Functions**.

While creating/editing a function, you’ll now see:

> **Connect Git Repository**

Click it → authenticate → pick your repo → choose a branch → set root directory and build steps.

Once connected, Appwrite will automatically deploy every time you push.



## 5. Test Everything

Make a tiny change inside your repo, commit, and push:

```bash
git push
```

Go back to the Appwrite Console → your function → **Deployments**.

You should see a new deployment triggered from your Git commit. If it’s green, everything works!



## 🐞 Troubleshooting

- **Auth errors:** Double-check values in `.env`  
- **Private key issues:** Re-encode or regenerate  
- **Deployments not triggering:**  
  - Localhost can’t receive GitHub webhooks  
  - Use ngrok if you want live push-triggered deployments  
- **Executor errors:** Ensure executor hostname matches your docker-compose file  



## 🎉 Final Thoughts

Setting up VCS once makes your entire development workflow *so much better*. No more manual uploads or repetitive packaging — just write code, commit, push, and let Appwrite handle the rest.

If you're working heavily with functions or sites locally, this setup is absolutely worth doing.

Happy building! 🚀
