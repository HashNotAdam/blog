---
title: "Ten minutes with YJIT"
date: 2023-06-09T09:00:00+10:00
slug: "ten-minutes-with-yjit"
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---

On 5 May, I gave a talk at [RORO Melbourne](https://www.meetup.com/Ruby-On-Rails-Oceania-Melbourne/) introducing YJIT and why you should care. The following is a recording of the talk and a transcript of the talk. Pick your poison!

{{< youtube JQmOIjrLruQ >}}

## Introduction

Hi, I’m Adam. For those who don’t know me, I’ve been engineer for a little over 2 decades and a Rubyist for about half that time.

I’m currently at Airclinix; a company I think you’re going to hear a lot more about. Airclinix is a next generation clinical platform, built by the healthcare community, to run tomorrow’s home visit, telehealth and clinic health services. But, I genuinely believe in the company vision, and so I’d say "watch this space".

## Ruby and Rails have you covered

I’ve never been more proud to be a Ruby developer. Thanks to amazing contributions from organisations like GitHub, Shopify, and Stripe, both Ruby and Rails have a really compelling story to tell in the enterprise space.

Whether it's:

- parallelism and concurrency via Fibers and Ractors;
- the type signatures and static analysis provided by RBS and RBI files and tools like Steep and Sorbet; or
- the Active Record support for more complex database configurations and strict loading for protection against n+1 queries

Ruby and the community is showing that you can have developer happiness and enterprise tooling.
And Rails 7.1 is looking like it could be the biggest minor release in Ruby on Rails history.

These are obviously not the kinds of features that people usually like to nerd out about—although, I did recently get a little too excited speaking to an SQL developer about common table expressions in Active Record—but these kinds of boring features are what allow us to focus on providing value. The only thing that really matters.

## Ruby 3x3

Back in 2015, Matz set a goal for Ruby 3 to be 3 times faster than Ruby 2. This was an ambitious goal that required considering performance from many different angles. One of those angles was to implement a Just in Time compiler, or JIT.

Now, the reason why this is a lightning talk is I figured, while a couple of people would be interested in how a JIT increases performance—most of you would be hating life. Given this, I’m going to oversimplify.

## Ruby needs to communicate

While we write Ruby in English, English is not the language that your computer understands so your code needs to be translated—what we call compiling.

The first step in the compilation process is Lexical Analysis where the code is broken down into its individual parts, such as keywords, identifiers, operators, and literals.

Then Syntax Analysis (or parsing) analyses the structure of the code to ensure that it conforms to the rules of the Ruby language.

Next, an Abstract Syntax Tree is constructed. The AST is a hierarchical representation of the code that can be easily traversed and manipulated by a machine.

The Semantic Analysis stage then checks the code for semantic errors and checks for proper usage of language constructs.

And, it’s at ths point, that it’s possible to compile the AST into bytecode which is run by YARV, the Ruby Virtual Machine.

Bytecode is quite efficient but it is still a platform-independent, higher-level representation. When your code is run, YARV converts the bytecode into machine code which is the binary representation that is specifically designed to work with the platform it is being run on.

Or, in other words, it’s the difference between generic instructions for any platform, and the specific instructions that, say, my Mac can follow.

The JIT adds an additional, asyncronous step to the process. It monitors which code is being run, looking for methods that are called frequently and it pre-compiles and stores the machine code in memory.

So, it’s "just in time" in the sense that it’s happening while the code is being executed but it isn’t really "just in time" in the sense that it has to warm up.

It’s going to take metrics and, based on those metrics, it’s going to decide in what areas it believes it can provide value.

If you want to get into the fine details of how your code is run, I would highly recommend [Ruby Under a Microscope](https://patshaughnessy.net/ruby-under-a-microscope). It’s been around a long time so it doesn’t get into JITs and some of the details have changed but it’s still a fantastic reference.

## YJIT

There have been a few JITs for Ruby, but the most exciting one so far is YJIT.

Created by Shopify, YJIT was added to Ruby as an experimental option in 3.1 but is now stable in 3.2.
It was originally written in C, but that had some serious drawbacks and so it was ported to Rust and became the first Rust in Ruby core.

This was discussed in some detail recently on [Remote Ruby](https://remoteruby.com/225) and I would recommend checking out that episode.

One implication of the transition to Rust is that you do need to have the Rust compiler installed before you compile Ruby. If you don’t have the Rust compiler, Ruby will still build but YJIT will not be installed.

This is pretty easy, though. On Unix-like systems like macOS or Linux, there is a single command on the [Rust Lang website](https://www.rust-lang.org/tools/install) which is all you need.

I also recently deployed to an Ubuntu server and I was able to add only the compiler to my Aptitude install and it really only took a minute to do.

If you want to check if YJIT is available, you can pass Ruby the enabled YJIT flag. In this example, Rust was not installed before Ruby and so you’ll see that YJIT isn’t referenced in the response.

```
$ ruby --enable-yjit -v
ruby 3.2.2 (2023-03-30 revision e51014f9c0) [×86_64-linux]
```

Whereas, in this case, because the Rust compiler was installed, YJIT was compiled with Ruby.

```
$ ruby --enable-yjit -v
ruby 3.2.2 (2023-03-30 revision e51014f9c0) +YJIT [×86_64-linux]
```

I was blown away when I enabled YJIT and saw a 1/3 decrease in server response time, which is an extraordinary improvement for something as simple as passing a flag.

Unfortunately, you can’t assume you’ll see the same thing and you absolutely should lean on your application performance monitoring.

If you’re not already on Ruby 3.2, make sure to perform that upgrade first because 3.2 has some lovely performance improvements and you’ll want to measure the JIT in isolation.

## That's it

For most of us, that’s it. Sit back and let the magic happen.

For organisations with code that is run at large scale, however, there could be value in optimising _some_ methods.

It’s generally a bad idea to adapt your code to the JIT—particularly one so new and likely to change—but the YJIT documentation does offer some guidelines.

For example, variables that change type make life difficult for JITs so you might avoid initialising a variable with, say, `nil` and then later assigning a string to it.

V8, the Google JavaScript engine, uses a JIT to speed up JavaScript execution. The V8 team have a fantastic blog and I really enjoyed [this post](https://v8.dev/blog/react-cliff) which got into the details of an issue in V8 that was uncovered by the React team.

In order to explain the problem, they need to get into some fine details about how V8, and JITs in general, work and the complexity in optimising in dynamic languages.

If you’re interested in understanding how to write code that’s optimised for JITs, that’s a great place to start.

You shouldn’t optimise for any specific JIT but, if you have the scale to justify it, there are some standard conventions you can follow.

I’m not working at that scale so I’m taking the easy win.
