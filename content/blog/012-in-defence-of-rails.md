---
title: "In defence of Rails"
date: 2022-06-11T15:50:00+10:00
slug: "in-defence-of-rails"
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---
I've been hearing a lot of sledging recently stating that design flaws in Rails
cause it to scale poorly to large organisations. Highlights include that it's
slow to boot because it loads everything upfront; that Active Record models make
it easy to create N+1 issues; and too much logic (including DB queries) end up
in views.

When designing a framework, authors need to decide what they are optimising for.
As is well described by Sid Sijbrandij in his article,
[Why We're Sticking with Ruby on Rails at GitLab](https://thenewstack.io/why-were-sticking-with-ruby-on-rails-at-gitlab),
DHH wanted to build something approachable but well structured.

I think it's hard to deny that Rails is both easy to get started with and it
allows you to get to market at lightning speed. Of course, being approachable
comes with some compromises. Most of the "getting started" tutorials I've seen
make a lot of choices I would never make in a production app. For example,
people are generally taught to put their business logic into models. Despite the
fact that some might consider me a geriatric engineer, I still remember the
many, many years feeling overwhelmed with all there is to learn. Given this, it
has always been my opinion that it's best to hide as much complexity as
possible until a newer developer has the space for the next concept.

This is not everyone's opinion. Frameworks like Hanami have made the choice to
drop you straight into the deep end. Their belief is that you will become a
better developer if you have no choice but to do things "right". There are also
people who believe you should be forced to learn SQL very early on so that you
know what your ORM is doing. While I respect their perspectives, my experience
is that there is a finite amount of information that anyone can take in and
trying to learn everything at once leads to someone who isn't very good at
anything; at least not for many years. You don't become a great engineer from
a few months of cramming; it's a life of continual learning. 

It's certainly true to say that if you take the approachable version of Rails
and scale it up to a large organisation, you're going to have a bad time. The
thing is, if someone has risen through to ranks to be in charge of engineering
at a large organisation should know better. None of the complaints I've heard
are enforced by Rails, they're just approachable defaults that are very simple
to change.

The following are some of the ways you can prepare Rails for scalability.

## Boot time

### Eager loading

In production, Rails will boot your whole application when it starts. This
ensures your users aren't made to wait while modules are booted during requests.
The default is to disable this in development and only enable it in test when
being run on CI.

Despite the defaults, I've seen apps with eager loading enabled for local
testing which means that even running a single unit test requires booting the
whole application.

It will depend on your application as to how much saving you'll get from
disabling eager loading but, in one of my applications, the difference in boot
time is about 20%.

### Rails helper

If you use RSpec, you'll know that it has a spec_helper.rb which contains the
settings needed to run tests on plain old Ruby objects but there is also a
rails_helper.rb which handles booting Rails and any settings that might be
needed to test Rails objects.

It's easy and convenient to place everything you need into the Rails helper, but
you need to remember that everything you put in there will be loaded regardless
of whether it's required for every spec.

When I optimised one Rails helper so it contained only what was needed for every
spec, it reduced boot time by 25%. The trade-off is that you will need to load
the additional settings when they are required but if you can't get test
feedback in a couple seconds (at worst) the feedback loop becomes too long. You
should want to run your tests frequently.

### Initializers

Even if you have eager loading disabled, it's likely that you are booting gems,
regardless of whether they are required, simply because you have an initializer
setting some config. Initializers is one area I haven't seen a clear solution
for and so, while I have ideas, I'm keen to crowdsource opinions.

In one application where boot times were getting a bit long, I simply disabled
any initializer that wasn't required in development or test. In those cases, I'm
using mocks and so there was no point in booting the gem when it would never be
used. This reduced boot time by about a third.

I've been doing some work in Hanami 2 and it leans very heavily on dependency
injection. Any class that is a child of a Hanami class can `include Deps`,
passing in a list of dependencies. While it's a different use case, it has me
wondering if a similar convention for lazy loading gem initalizers could be
valuable. Given the added verbosity, most likely you'd want to check the real
impact on boot time per application and decide if it's worth it. I have
typically found that initializers can be a big culprit for slow boot times but
other approaches might be better suited to your app.

If you have other ideas, I'd love to hear them.

### Minimal

If you're building a small application with minimal dependencies, e.g., a
microservice, you probably don't need to bring in all of Rails. When you create
a Rails app, you can skip anything you don't need or, if you really only want
Rails to provide some structure and consistent conventions, you can create a
minimal build.

```bash
rails new --minimal --skip-active-record --skip-asset-pipeline --skip-test my_application_name
```

That command will leave you with models, views, and controllers but none of the
extras you would normally expect. If you'd like to include some extras, run
`rails help` and curate your own selection.

If you have an existing application with dependencies you don't need, you can
easily disable them in `application.rb`. In the case of something like Active
Record, remember to also remove the database dependency from you Gemfile.

In a tiny application, you may not see huge gains from this thanks to lazy
loading in test and development and the caching provided by Spring. Saying this,
it can be valuable to keep a microservice clean and focused.

## Code cleanliness

### ViewComponent

Created by GitHub, [ViewComponent](https://viewcomponent.org/) is "a framework
for creating reusable, testable & encapsulated view components, built to
integrate seamlessly with Ruby on Rails."

Essentially, it solves the issue of business logic and DB queries in the view.
Components include a Ruby object for your logic, a test/spec for testing both
your logic and your component's view in isolation, a template, and optionally,
component specific assets. This makes components completely self-contained
and highly testable. Since you can unit test your logic without rendering the
view, it makes your tests substantially faster. It doesn't remove the need for
rendered tests but it allows you to minimise them and render only that
component, not the whole page.

### Command objects

A common way to organise business logic is the command/query pattern. Commands
are classes that handle a single task that changes state and they should be
given a name that reflects that action (e.g., `CreatePost` or
`CalculateCharge`).

It's easy enough to create your own implementation of the pattern but there are
also helpful gems that can take away the boilerplate. My preference is
[ActiveInteraction](https://github.com/AaronLasseigne/active_interaction).
ActiveInteraction is an implementation of the command pattern, but they refer to
commands as "interactions". You, however, can use whatever name suits you.

I appreciate that interactions have typed inputs, but it also utilises Active
Model Validations which helps maintain a consistent validation style across your
application. When you have a hash input, you can choose to fully type the hash
values and let ActiveInteraction strip out any unapproved keys.

The [ActiveInteraction::Extras](https://github.com/antulik/active_interaction-extras)
gem includes a whole range of valuable helpers including the ability to offload
handling of Strong Parameters to your interactions. Handling parameter filtering
in my controllers has never felt quite right to me. I like my controllers to be
simple paper-pushers with all the implementation details hidden elsewhere. When
I'm trying to understand some code, I want to be able to quickly skim through an
overview of the process and only dig into the implementation details if it's
required.

Because ActiveInteraction embraces the Active Model conventions that you're used
to, an instance of an interaction has methods like `valid?` and any validation
errors are added to an ActiveModel::Errors object available by calling `errors`,
just like when you work with an Active Record model.

Every interaction has an `execute` method as the entry point and you commence
the interaction by sending either a `run` or `run!` message. Calling `run` will
return you the instance with `success?`, `failure?`, and `result` methods.
`run!`, on the other hand, returns the result or raises an exception if there is
an error. Interactions are also highly composable and, if you are using `run`,
errors are passed back up the chain so you can deal with them gracefully.

Leaning into that composability allows you to ensure that each interaction is
only ever handing a single responsibility. I've found the combination of the
interaction interface and maintaining single responsibilities has heavily
simplified my testing. In fact, if I ever think "I wonder how I can test this",
it's almost certainly a sign that there is a missing abstraction.

### Query objects

Queries encapsulate your interactions with the database without fattening up
your model. Unlike commands, queries shouldn't have side effects and, when
using Active Record, they should generally be composable.

It was great to see
[Thiago Araújo Silva from thoughtbot](https://thoughtbot.com/blog/a-case-for-query-objects-in-rails)
considering ways to blend the value of Active Record scopes with query objects.
Whether or not you agree with their pattern, the article is great for driving
the kinds of discussions that can uncover what is important to your
organisation.

### Strict loading

A common complaint with Active Record is that it's easy to create N+1 query
issues. There multiple gems designed to uncover these—the most popular of which
is [Bullet](https://github.com/flyerhzm/bullet)—but the only thing better than
fixing N+1 queries is not creating them in the first place. This is where strict
loading comes into play.

By adding `strict_loading: true` to your Active Record associations, Rails will
not allow you to call upon any association that hasn't been eager loaded. For
example:

```ruby
class Post < ApplicationRecord
  has_many :comments, strict_loading: true
end

Post.last.comments.each
# ActiveRecord::StrictLoadingViolationError: `Post` is marked for
# strict_loading. The `Comment` association named `:comments` cannot be lazily
# loaded.

Post.includes(:comments).last.comments.each
# This is fine
```

I quite often use the Rails console to get a feel for the data I'm working with
and find strict loading to get in the way so I have it enabled everywhere except
in the console.

```ruby
class Post < ApplicationRecord
  has_many :comments, strict_loading: strict_loading?
end

class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  def self.strict_loading?
    $PROGRAM_NAME != "rails_console"
  end
end
```

If there is a reason why enabling strict loading at the model level won't work
for you but you're debugging an N+1, you can also enable it just for one query.
If you don't have the strict structures discussed above in place, it may be hard
to track down where the N+1 is occurring. Enabling strict loading at the entry
point should cause and exception at the N+1 which (hopefully) you can uncover
with a simple run of your test suite.

```ruby
post = Post.strict_loading.last
...
post.comments.each
# ActiveRecord::StrictLoadingViolationError: `Post` is marked for
# strict_loading. The `Comment` association named `:comments` cannot be lazily
# loaded.
```

### Domains

As an application grows, it can be difficult to navigate and understand the
boundaries if all your files sit alongside each other. This becomes even more
of a concern when there are many teams that each have ownership of different
parts of the project.

While some organisations use tooling to map teams to files, I prefer to separate
my application into clearly defined domains. This, of course, assumes that your
organisation assigns team to domains.

At first, it might seem that you need to follow the Rails convention of placing
your models, views, and controllers in the automatically generated directories
under `app`, but there is no reason why you can't have a set of those
directories within every domain. For example:
- app
  - domains
    - first_domain
      - assets
      - controllers
      - models
      - views
    - second_domain
      - assets
      - controllers
      - models
      - views

The compromise is a bit of verbosity since all your classes will be namespaced.
Having been exposed to years of Java development, I don't accept verbosity
without careful consideration, but I find that this allows me to navigate an
application with more clarity and it's easy to see the boundaries of each
domain. If your domains also map to teams, you know immediately what you are
free to modify and what is going to require consultation.

### Coupling between modules

Even with your application grouped by domain, you might find that you are
lacking clear boundaries. If domains are allowed to call any class within
another domain, the domains become tightly coupled and velocity plummets as
teams need to coordinate on changes. Smaller organisations can often get away
with enforcing conventions but, in larger organisations, it might be necessary
to lean upon tooling.

Shopify's solution was to create the
[packwerk](https://github.com/Shopify/packwerk) gem which enforces strict
boundaries between code groups. You categorise your classes as either public
or private. Classes within the group can send each other messages but classes
from other groups can only interface with public classes. By creating these
boundaries, you get all the benefits of a monolith without hindering
productivity.

While I support certain use cases for microservices, I'm convinced that most of
what has been driving the movement is organisations creating a
[big ball of mud](https://en.wikipedia.org/wiki/Big_ball_of_mud) and then
blaming the monolith for their lack of application architecture.

Another solution I've heard of is splitting your monolith into multiple Rails
Engines. With separate Engines, it's impossible not to see the boundaries of
responsibility and there is increased scope for teams to have differences in
their styles. On the other hand, it does increase complexity when you need to
communicate between domains.

## Conclusion

Even though I'm a big fan of the Rails way, I'm excited that projects like
Hanami offer a different methodology for those with differing views. There is
plenty of space for differing styles and there are some interesting patterns in
there that we can all be inspired by.

I would hate for anyone working on a sub-optimal Rails project to believe that
is just what working with Rails is like. As someone who is outcome driven, I
get a lot of joy working on a well optimised Rails project. Importantly, I've
been able to accomplish more with scarce resources than many large, well-funded
teams by leaning into the simplicity but applying the appropriate guard rails.

If you're working on a project with some of these concerns, you can do something
about it. If you're dealing with years of neglect, it might take time, but you
can chip away at the problem and make your life continually better. Ruby is all
about developer happiness, after all.
