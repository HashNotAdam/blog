---
title: "Cards Against Isolation (Installation)"
date: 2020-08-13T16:30:10+10:00
slug: "cards-against-isolation-installation"
aliases:
  - /blog/cards-against-isolation-installation
description: ""
keywords: []
draft: false
tags: [development, tutorial]
math: false
toc: false
---
This part is just for those people who are new to Ruby and either haven't yet
install Ruby and PostgreSQL or just want to see how I do it. If you're all over
this, you can skip to the [next step](/blog/cards-against-isolation-setup);
just make sure you have a version of Ruby 2.6 installed.

I'm afraid this part of the tutorial is macOS only since that is where I'm
comfortable. The components are still relevant if you are on Linux but, if
you’re a Windows user, there could be important differences.

## Editor

Your editor (the application in which you write your code) is a bit beyond the
scope of this tutorial. Hopefully I'll write a post in the future explaining
what I love about Visual Studio Code and how I configure it for Ruby but, for
now, I'm afraid you are going to need to seek advise elsewhere.

UPDATE: [@pmorren](https://twitter.com/pmorren) has made me aware of
[Make VS Code Awesome](https://makevscodeawesome.com/) which looks like a
seriously comprehensive look at how to use VS Code like a pro.

## Homebrew
Homebrew is the best and most used package manager for macOS—basically, it finds
and installs applications and tools for you. We’ll use it to install PostgreSQL
and the tools we need to get a version of Ruby up and running. You should check
the latest instructions on the [Homebrew website](https://brew.sh/) but, at time
of writing, you simply need to open your Terminal app and run:
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

## Ruby environment manager
The Ruby language is continuously evolving and that can have implications for
how your code runs in the future (or if it runs at all). This is why it’s
valuable to have a Ruby environment manager—a tool that allows you to have
multiple versions of Ruby installed at the same time and specify which version
should be used in each project. There are a few of them around but my favourite
is [rbenv](https://github.com/rbenv/rbenv). It’s a simple tool that doesn’t
muck around with your system or Ruby libraries (unlike its big competitor, RVM).
rbenv itself can’t install new versions of Ruby for you but it integrates with
ruby-build to provide that functionality.

rbenv can be installed via Homebrew by running:
```bash
brew install rbenv
```
This will also install ruby-build for you.

Once installed, you just need to start rbenv. It will create a .rbenv directory
in your home folder which is where the versions of Ruby and any gems (libraries)
will be installed. This keeps everything isolated to just your user and does not
effect anyone else who may use the same computer.
```bash
rbenv init
```

Calling the init command every time you want to write Ruby code is a bit of a
pain in the arse. When you call init, rbenv will give you instructions on how to
get it to automatically boot. In my case, I need to add the following to
`~/.zshrc`:
```bash
eval "$(rbenv init -)"
```
It's possible that this file won't exist yet, but that's okay, running the
following command will either append it to the existing file or create the file.
Then the second line will just tell the shell to run the configuration file
(this is done automatically when you start a new tab or window):
```bash
echo -e '# Ruby environment manager\neval "$(rbenv init -)"' >> ~/.zshrc
. ~/.zshrc
```
You may have been told to change `~/.bash_profile`; if so, just replace `zshrc`
with `bash_profile`

## Ruby

We're going to use Ruby 2.6, the latest version of which is 2.6.6 at time of
writing. While Ruby 2.7 has been released, it introduces deprecation warnings
that aim to help prepare you for changes in Ruby 3. Those warnings can be pretty
annoying. While Ruby on Rails has fixed these future issues in verison 6.1, it’s
possible you will use a gem that hasn’t been updated yet. There will be enough
for you to think about while building this game without also navigating obscure
messages coming from someone else’s code.

Now that you have rbenv, installation is simple:
```bash
rbenv install 2.6.6
```
This can take a while so you might want to consider moving to the next step and
then coming back here.

Once installed, you can tell rbenv to use the new version rather than the system
installed version with:
```bash
rbenv global 2.6.6
```

I've found that, after first installing rbenv in the zsh shell, I needed to run
`eval "$(rbenv init -)"` after the first time I set the global. If you run
`ruby -v` and you don't see something that begins "ruby 2.6.6", then you also
re-run that command.

## PostgreSQL

Thanks to Homebrew, this is extremely straight forward:
```bash
brew install postgres
```
If you've skipped ahead while waiting for Ruby to compile, you can run this in a
new tab by pressing command-t while in your terminal.

Once it's installed, you need to start it. This command not only starts Postgres
but it makes sure it starts on boot:
```bash
brew services start postgresql
```

One thing you should know is that Postgres data files are generally not
compatible between versions. If/when you upgrade in the future, make sure you
read the instructions Homebrew gives you. They will always give you a command
you can run that migrates your data from the old version to the new version.

## Yarn

Finally, Yarn is a package manager for JavaScript which saves you searching for
and manually downloading every JavaScript library you require and ensures the
correct versions of those libraries are maintained.

Yarn can also be installed via Homebrew:
```bash
brew install yarn
```

Next, we need to [setup the project](/blog/cards-against-isolation-setup)
