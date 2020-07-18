---
title: "Deploying this blog to Netlify"
date: 2020-07-18T20:22:22+10:00
slug: ""
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---
To continue my run of being pleasantly surprised by modern tooling, I decided to
try out Netlify since it appears to be all the rage these days. As with creating
the blog itself, hosting it on the web was a mere 5-minutes effort and
completely free (unless it turns out I'm not actually the bore I suspect I may
be and this blog goes viral).

After signing up, all it took was pressing the “New site from Git” button. From
there I could connect my Github account and select the repository I created for
this blog. I was shocked to see that Netlify not only connected the repository
but it worked out the site was built with Hugo and automatically set the build
command to `hugo` and publish directory to `public`.

One really important note is that Netlify seems to fail to build unless you
include a configuration file. Thankfully, Hugo
[provides a configuration template](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/)
which worked for my deployment without modification.

The whole experience was extremely pleasant. I give Hugo and Netlify ⭐️⭐️⭐️⭐️⭐️
