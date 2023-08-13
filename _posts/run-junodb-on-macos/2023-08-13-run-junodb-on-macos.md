---
title: Run JunoDB on mac
date: 2023-08-13
tags: ['paypal', 'junodb', 'dev']
description: How to setup and run PayPal JunoDB on your mac machine?
---

A few months ago, Paypal's home-grown key-value store [JunoDB](https://github.com/paypal/junodb/tree/dev) was made public. Being a dev I want to try it in my local and I found it a bit difficult to setup because of lack of proper documentation. For exmaple: looking at the repository it looks like repo is built based on OSS technology but when I try to setup locally I'm getting an error as shown later in the screenshot and I need to login docker 😤 why??? I decided to create an account and login docker to perform docker pull by one of the command `docker/build.sh` but I got the same error as previous.

<img src="/assets/img/run-junodb-on-mac/run-junodb-mac-error.png" alt="JunoDb build error" width="100%" height="100%" />

```console
Error response from daemon: Head "https://ghcr.io/v2/paypal/junodb/junoserv/manifests/latest": unauthorized
```

After fidling through the code and build script I found another docker pull registry which was commented here
[https://github.com/paypal/junodb/blob/dev/docker/build.sh#L32](https://github.com/paypal/junodb/blob/dev/docker/build.sh#L32) after uncomment this line and commenting the next line in the script i.e. line 33. I ran the same step `docker/build.sh` it started to pull docker images and started compiling things and ultimately after sometime build was successfull.


<img src="/assets/img/run-junodb-on-mac/run-junodb-mac-diff.png" alt="JunoDb build script changes" width="100%" height="100%" />


Successfull build
<img src="/assets/img/run-junodb-on-mac/run-junodb-mac-successful-build.png" alt="JunoDb build success" width="100%" height="100%" />

After that I ran the next script mentioned in the `README.md`
```console
docker/start.sh 
```
Again it took sometime to create the container and once it was done I was able to test all the commands that are mentioned in the readme. I was able to `create` `get` values from the JunoDB.


<img src="/assets/img/run-junodb-on-mac/run-junodb-mac-testing.png" alt="JunoDb test commands" width="100%" height="100%" />

Outro: In the next coming post I will try to write my own client in either JS/TS or Rust programming language. I have not yet decided but for the learning purpose I will explore this area. Stay tuned.
