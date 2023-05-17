---
title: "Starting a blog"
date: 2020-07-18T19:01:05+10:00
slug: "starting-a-blog"
aliases:
  - /blog/starting-a-blog
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---
It feels somewhat unusual to be starting a blog two decades into my career but
if there is anything I've learnt on this journey it's that the learning never
ends. I may have well and truly missed peak-blog but it seems that now is as
good a time as any to share what I know.

Creating this blog required no more than 5-minutes of effort. It may not be the
most breaktaking product you've seen but it's pragmatic.

[Hugo](https://gohugo.io/) is a static site generator. Installing it on macOS is
simple with Homebrew:
```
brew install hugo
```
Once installed, it's easy to create a new site:
```
hugo new site my-new-site-name
```
Make sure you `cd` into your new directory:
```
cd site my-new-site-name
```
And then, if you're like me and would rather not spend weeks designing and
building a site theme, you can add one of the
[existing themes offered by Hugo](https://themes.gohugo.io/) (I chose Codex):
```
git submodule add https://github.com/jakewies/hugo-theme-codex.git themes/hugo-theme-codex
```

The themes include some configuration items that you need to copy to your
`config.toml` file. This is super easy, just follow the directions on the
theme's page. Importantly, there will be a line that enables the theme like:
```
theme = "hugo-theme-codex"
```

You now have a themed website with just 4 commands. At this point, I spent a
moment wondering where the trick was—development is rarely this simple.

Run the server to see your site and, by default, you can see it at
[http://localhost:1313/](http://localhost:1313/):
```
hugo server -D
```

To add a page to your blog, run:
```
hugo new blog/name-of-your-post.md
```
This is add a page to `content/blog` which uses markdown. For example, this post
looks something like this:
```
---
title: "Starting a Blog"
date: 2020-07-18T19:01:05+10:00
slug: ""
description: ""
keywords: []
draft: true
tags: [development]
math: false
toc: false
---
It feels somewhat unusual to be starting a blog two decades into my career but
...
```

When your post is ready to go live, just make sure you change the `draft`
parameter to `false`
