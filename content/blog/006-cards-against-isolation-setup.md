---
title: "Cards Against Isolation (Setup)"
date: 2020-08-20T13:24:20+10:00
slug: "cards-against-isolation-setup"
description: ""
keywords: []
draft: false
tags: [development, tutorial]
math: false
toc: false
---
Now that Ruby, Postgres, and Yarn are installed, we can setup a new project.

## Installing Rails

In Ruby, libraries of code that are packaged and distributed for everyone to use
are called gems. One way these gems can be installed is via the `gem` command
which connects to the [RubyGems](https://rubygems.org/) repository to download
the library you are looking for. Rails is distributed as a gem and so we first
need to grab a copy of it from RubyGems.

It's a good idea for us to synchronise our versions of Rails so there aren't
differences between the syntax I use and what Rails is expecting. Given that,
the command is:
```bash
gem install rails -v 6.0.3.4
```

In the same way your application will depend upon Rails, Rails also depends upon
other gems. The `gem` command knows this and will grab everything needed for
Rails to work.

While most gems are simply downloaded and installed, other gems will include
code written in C. Gem developers might use C when they have specific
performance requirements that justify the effort. Ruby is really fast but there
are many tasks where the speed difference between C and Ruby matters. One
example would be interacting with your database. It is vital that your database
queries are blazingly fast and so you can expect those gems to have a lot of C
code. You’ll notice the gems with C bindings because you’ll see “Building native
extensions. This could take a while…”. Occasionally you’ll try to install a gem
and you’ll find one of those little buggers will fail. Unfortunately, the errors
are generally obscure and you just need to get good at finding the important line
in the error message (even if you don’t understand it) and popping it into
Google. You can be quite certain that you won’t be the first person to see the
issue so don’t even bother trying to debug it yourself.

I've been following this tutorial in 2 completely new macOS virtual machines,
one High Sierra and one Catalina, and I encountered an error when Bundler tried
to install Puma. To get around that issue I ran:
```bash
bundle config build.puma --with-cflags="-Wno-error=implicit-function-declaration"
```
and then called `bundle` again. If this doesn't work for you, Google is your
friend.

## Create the project

Before creating your project, you'll need to decide where you want this code to
live on your computer. There is no right or wrong answer to this other than to
say that your life will likely be made a lot easier if the answer is somewhere
in your home directory.

In my case, I just put it in a folder called "dev" at the root level of home.
Using your terminal, you will want to change to this directory using `cd`. In my
case, the command is:
```bash
cd ~/dev
```
Where `~/` just means "in the home directory". Since I'm on a Mac, `~/` is the
same as `/Users/arice/` but it's both less characters to type and also the same
across users and computers.

The command we'll use to create the project directory is:
```bash
rails _6.0.3.4_ new cards_against_isolation \
  --database=postgresql \
  --skip-action-mailbox \
  --skip-action-text \
  --skip-active-storage \
  --skip-turbolinks \
  --skip-test \
  --skip-system-test
```

While that is working away—it will take a bit if this is your first Rails app
on this version of Ruby—let’s break down what that all means.

`rails _6.0.3.4_`: normally you wouldn't see that second part. The `rails`
command automatically installs the latest version available, however, I want to
be sure we're both on the same page so I'm explicitly telling it which version
to use.

`new cards_against_isolation`: create a new project called
cards_against_isolation.

`--database=postgresql`: set up Rails to use the PostgreSQL database. This will
install the `pg` gem in your application (we'll discuss how this works soon).
If you are using MySQL, you can simply replace this with `--database=mysql`.

By default, Rails will install all the available features. The approach is to
make it as easy as possible to get started and then allow you to modify things
as you learn more about it. Given this, we are turning off the features we won't
be using.

`--skip-action-mailbox`: [Action Mailbox](https://guides.rubyonrails.org/action_mailbox_basics.html)
allows Rails to accept and process incoming emails. While we will want to send
emails, we have no need to receive them.

`--skip-action-text`: [Action Text](https://edgeguides.rubyonrails.org/action_text_overview.html)
gives you text input fields that can allow rich text (think bold, italics,
bullet points, and links to websites). We only need people to enter basic
information like their names. The players are already going to be bold in their
actions, we don't also need them to be bold in their typography.

`--skip-active-storage`: [Active Storage](https://edgeguides.rubyonrails.org/active_storage_overview.html)
can be handy when you want people to be able to upload files.

`--skip-turbolinks`: [Turbolinks](https://github.com/turbolinks/turbolinks) is
an interesting tool for speeding up page loads. When a page change occurs
(for example, when someone clicks a link), instead of reloading the page,
Turbolinks replaces the body of the page and loads any JavaScript or CSS that
wasn't previously loaded. This can have huge performance implications because
those scripts and styles can take a bit of time for the browser to parse. On the
other hand, the fact that the page isn't reloading causes "quirks" that you need
to account for. While Turbolinks provides plenty of ways to navigate these
concerns, I haven't found the benefits to outweigh the costs in a small
application like this. I do have production applications using Turbolinks,
however, and in the right app it can give you some stunning results.

`--skip-test` and `--skip-system-test`: Rails uses Minitest as its testing
framework. There is nothing wrong with Minitest; it is beloved by many who argue
that the use of plain old Ruby code makes it is easy to learn. I have a strong
preference for Rspec which nudges you towards telling a story. For people new to
Ruby, it can be a little more work to understand but I believe testing in
narrative not only helps you think about what you are testing but, when reading
the tests, you can easily understand both what the application does and what is
being tested. We'll add Rspec later.

Now, with everything installed, we need to change to the new directory:
```bash
cd cards_against_isolation
```

One interesting thing to note is that the Rails install will add a file called
`.ruby-version` which contains nothing more than version of Ruby used to create
the project. As time progresses and you use newer versions of Ruby for newer
projects, this file will tell your Ruby environment manager to change version.
My global version of Ruby is 2.7.1 but, as soon as I `cd` into this project,
rbenv automatically switches from 2.7.1 to 2.6.6.

### MySQL installation error

While this tutorial (and I) prefer PostgreSQL, I did test this process using
MySQL and ran into the following issue:
```
linking shared-object mysql2/mysql2.bundle
ld: library not found for -lssl
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

The solution to errors while compiling native binaries is different depending on
the very specific language  of the error but, in this case, I solved the problem
by running the following:
```bash
brew install openssl
export LIBRARY_PATH=$LIBRARY_PATH:/usr/local/opt/openssl/lib/
```

## Your first commit

When building any application, there is no question you are going to make
mistakes—I've never encountered a developer who has reached a state of
enlightenment—and I expect to make mistakes in this build. To manage this known
issue, we use a Version Control System (VCS) to keep track of our changes. These
days, there is really only one VCS you need to know about, git. If you haven’t
used git yet, I’d recommend you go and sign up for an account at
[GitHub](https://github.com/) now. Services like GitHub will store your code in
the cloud. Any projects you allow to be public can be seen by anyone and can
become part of your CV. A git repository also allows multiple people to work on the
same project. You don't actually need to store your code in the cloud—it's
absolutely possible to track your changes only on your computer—but if nothing
else, it acts as a handy backup.

The Rails installer actually creates a new git repository for you; all you need
to do is create your first “commit”. A commit saves your changes to the
repository.

We’ll get into good git practices later but, for now, you first need to “stage”
all the files for commit. Staging tells git which files (or changes in files) to
store in the commit.

This stages every file in the current directory:
```bash
git add .
```

Next you need to create the commit and write a message explaining what you did.
If you are new to git, you are going to want to make sure your shell knows to
which editor you use. If you are not used to terminal editors like vim, you're
going to have a terrible shock when trying to write your commit message.
Unfortunately, the solution to this problem varies by editor.

You can check if an editor is set by running:
```bash
echo $EDITOR
```
and seeing if anything is returned. If you just get an empty line, you are going
to want to set this.

If you are using VS Code, firstly make sure you have the `code` shell command
installed. If you’ve ever typed “code” in your terminal, you have this. If not,
open the command palette (command-shift-p or View > Command Palette) and type
“shell”. One of the options will be to “Install ‘code’ command in PATH”.

Next, you can set VS Code as your editor by running:
```bash
CONFIG_FILE=`if [ $0 = "-bash" ]; then echo "$HOME/.bash_profile"; elif [ $0 = "-zsh" ]; then echo "$HOME/.zshrc"; else "Shell unknown"; fi`
echo -e "\nexport EDITOR='code --wait'" >> "$CONFIG_FILE"
. $CONFIG_FILE
```

Now, running:
```bash
git commit
```
should load the commit window in your editor ready for you to provide a message.
You can think of the first line of a commit message as a subject while
subsequent lines serve as the message body. You want your message to convey what
changed but it’s more important to explain why the changes were made.

Given all we have done is install Rails for a personal game app, I think it’s
acceptable to break the rules a little and simply write:
> Install Rails

Make sure you save and close the commit window. You should expect commit
messages to become more descriptive once we are making decisions that aren’t so
stupendously mind-numbing.

## Push to the repository

Now is a great time to configure our git repository. If you don’t intend to save
this code in the cloud you can skip this step but I would strongly recommend you
push everything up.

Before pushing, we should change the name of our branch. By default, git will
name your primary branch "master". This terminology was inspired by enslavement
and it best we move past it. Unfortunately, see language like this a lot in
technology including master/slave database configurations and white/blacklists.
Thankfully, it’s easy to change our branch name to "main":
```bash
git branch -M main
```

Whether you use GitHub, Bitbucket, or another provider, the process of getting
your code to the cloud is pretty much the same. First you need to create your
new remote repository. This can be done via their web interface or many of the
providers also have desktop apps.

You’ll need to give your remote repo a name. This will become part of the URL so
pick something meaningful. GitHub has suggested that, if I need inspiration,
`furry-waddle` might be a good option. While I appreciate this fine suggestion,
I’m going to stick with `cards-against-isolation`. If you’d prefer to call yours
`furry-waddle`, that’s totally fine so long as it is meaningful to you.

You’ll probably be given 2 URLs, one HTTPS and one SSH. While HTTPS is the
easiest to get started with, you’ll be forced to authenticate every time you
want to push new code which is at least as annoying as it sounds. If you haven’t
already added an SSH key to your version control provider, I’d recommend you go
away and sort that out now. It’s a few minutes of pain for years of simplicity.
[Here are the GitHub instructions for macOS](https://docs.github.com/en/github/authenticating-to-github/connecting-to-github-with-ssh).

To add your remote repository URL to your project, run:
```bash
git remote add origin [your-url]
```
For example, my command is:
```bash
git remote add origin git@github.com:HashNotAdam/cards-against-isolation.git
```

This has now told git where to send the code but it is still necessary to
point the current local branch to a branch on the remote repository. It is
pretty much always the case that the name of your local branch is the same as
the one on the remote (it's super confusing if that isn't true) but git still
needs you to explicitly tell it that. Thankfully, you never need to remember the
command to make that connection—while I’m sure I could remember it if I tried, I
have no reason to try—just attempt to push your code and git will tell you what
to do. If you run:
```bash
git push
```
you will be told:
```bash
fatal: The current branch main has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream origin main
```
Not surprisingly, you can copy and paste that command. You only need to do this
once per branch. When you are working with others, you should really be creating
a new branch for every feature where a feature should (ideally) be small enough
to complete within a day. For this, we will only use the main branch.

Once you've run that command, your code is safely in the cloud and we can start
doing something substantial [in the next post](/blog/cards-against-isolation-authentication)
