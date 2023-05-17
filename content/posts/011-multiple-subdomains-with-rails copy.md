---
title: "Multiple subdomains with Rails... and Australian domains"
date: 2021-10-02T10:53:14+10:00
slug: "multiple-subdomains-with-rails"
description: ""
keywords: []
draft: false
tags: [development]
math: false
toc: false
---
Ruby on Rails has come a long way. It's always been a powerful framework but,
thanks to the likes of GitHub and Shopify, it now includes advanced features
like out-of-the-box WebSockets, parallel testing, and multiple databases with
sharding. Given the recent push into enterprise functionality, I've been
surprised by how poor my experience has been using multiple subdomains.
Hopefully I can save you some pain.

One of the many services provided by Education Advantage is repairs on Apple
devices. Back in 2014 I made my first Rails app; a place for customers to log
and track repairs. Since then, our focus has been mostly on internal automation
rather than customer-facing applications. This year, however, we decided to
start expanding our external presence. The first new project is essentially a
help desk crafted specifically for our market with integrations around the
business.

It would have been quite easy to add some new controllers to the application and
expand the existing menu, however, I see repairing devices and a help desk as
being fundamentally different concepts with wildly different interfaces and this
is only the first of many new ideas. I've seen a lot of platforms that try to
package the kitchen sink into one app and I can't recall ever feeling like it
did anything other than create unwieldy navigation and a poor user experience.
(Yeah, I see you AWS console)

I made the decision that keeping the current system working as it is on the
"service" subdomain and creating a new "support" subdomain would keep things
clean and focused. I also intend to later add a "dashboard" that ties everything
together.

The
[Rails routing guides](https://github.com/rails/rails/blob/v6.1.4.1/guides/source/routing.md#request-based-constraints)
makes this sound easy enough; just wrap your routes in a subdomain constraint
and let the magic happen.

Based on that documentation, I started with something that kind of looked like
this... but bigger:
```ruby
Rails.application.routes.draw do
  devise_for :users

  constraints subdomain: "service" do
    root to: "jobs#index"
  end

  constraints subdomain: "support" do
    root to: "tickets#index"
  end
end
```
I want users to be able to update their profile and their password regardless
of the subdomain they are on, but I only want Jobs to be on the "service"
subdomain and Tickets only on "support".

## Duplicate routes not allowed

It turns out you can't have multiple routes defined as "root". While this can be
confusing if you think about it from the perspective of paths (the root URL will
be `/` regardless of subdomain), there would be no way for Rails to know which
subdomain you wanted if you asked for the `root_url`.

This can be solved by giving custom names to each of the routes:
```ruby
constraints subdomain: "service" do
  root to: "jobs#index", as: :service_root

  resources :jobs
end

constraints subdomain: "support" do
  root to: "support/tickets#index", as: :support_root
end
```

If you use `root_path` anywhere in your application, just be sure to replace
that with the new helper method.

## Localhost isn't gonna fly

This is the issue that probably frustrated me the most because it seems like
something that should be clearly documented but I uncovered it via Stack
Overflow. You can't use localhost when working with subdomains, you'll need to
use a "real" domain.

There are services that offer domains that simply point back to your device but
I chose not to rely upon external providers and instead modified my host
records.

Operating Systems based on Unix (including macOS) generally include a file,
`/etc/hosts`, that facilitates a kind of personal DNS. When you make a web
request, the OS checks if it has a record for that domain/IP before passing the
lookup up the chain. This means you can override the IP of any domain by adding
to this file. The file requires super user privileges to edit so I modify it
with Vim (`sudo vim /etc/hosts`):
```
##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost

127.0.0.1 service.easi.dev.au
127.0.0.1 support.easi.dev.au
```
Yours might look a bit different but all that matters is those last lines are in
there somewhere.

127.0.0.1 is an IP address that always refers to your local device so it's
telling the operating system that request to `service.easi.dev.au` and
`support.easi.dev.au` should be handled by my laptop.

## Tell Rails the host is okay

Rails will reject any domain that it isn't expecting so you need to modify your
development environment to include the host.

`config/environments/development.rb`
```ruby
Rails.application.configure do
  config.hosts << ".easi.dev.au"
```

If your Rails server is running, you'll need to reboot it for that configuration
to be read.

## Cross-subdomain sessions

By default, Rails assumes there is only 1 part to the TLD. This is great if you
happen to be American and using a `.com` but is highly problematic if, say, you
live in Australia and use `.com.au`.

The way Rails handles the number of parts in a TLD is both poorly documented and
highly confusing.

Firstly, for Rails to be able to parse the URL and handle routing appropriately,
you need to tell it the "TLD length". In the case of Australian `.com.au`
domains, the `com` is 1 and the `au` is 1 so the length is 2.

In `config/application.rb`, add:
```ruby
config.action_dispatch.tld_length = 2
```

I've added this to `application.rb` because I find it easier to use the same TLD
length in development, test, and production. If you have different TLD
lengths—for instance, `.fake` in development and `.com.au` in production—then
you will want to add this setting and the appropriate lengths to your
environment configs (`/config/environments/[development,test,production].rb`).

Now your site should be loading across your subdomains, however, the session
will only be valid for the subdomain you logged in on. As soon as you click a
link that takes you to the other subdomain, you will be navigating as an
unauthenticated user.

If you inspect the cookies for the site in your browser developer tools, you'll
see the cookie domain is set to the full domain of the site you logged in on
(in my case, `service.easi.dev.au`) meaning it is not valid on any other domain
or subdomain.

Create a new initializer in `/config/initializers/session_store.rb`:
```ruby
Rails.application.config.session_store(
  :cookie_store,
  key: "_easi_session",
  domain: :all,
  tld_length: 3
)
```

Now, you may be wondering why the `tld_length` length is 3 here when we only
just discussed that it should be 2. Yeah, well, so am I and I'd rather not
reopen the box I've used to encase all the rage I felt while debugging this.
I've moved on and I simply add 1 to whatever the number should be 🤷‍♂️

Be sure to reboot your Rails server.

## Aside for those with a TLD length of 1

Like many devs, I've been using the `.dev` TLD for my local development for many
years. Unfortunately, Google bought this TLD and now enforces that all sites
have a secure connection. Since I don't want to set up SSL/TLS certificates in
development, I've stopped using it.

If you are developing a site for a `.com` or similar TLD and don't what to use
something like `.dev.au`, keep in mind you can pretty much use whatever you want.
For example, `website.fake` or `example.pickles_make_me_chuckle`.

## Testing

If you are like me, you might get to this point and celebrate your
accomplishments. Before you can commit you need to run your tests to make sure
everything still works. Of course, you celebrated too soon because yours tests
are now red all over.

This seems simple enough, you just need to tell the tests which subdomain to
use... right

![](/blog/011-multiple-subdomains-with-rails/Anakin-Padme.jpg)

I test with Rspec so the details will be different for Minitest or any another
framework. For your request specs and non-JS feature specs, it is possible to
simply change all of your requests from `something_path` to `something_url` and
let Rails add the appropriate subdomain. The default domain used for non-JS
tests is `www.example.com` so those become `subdomain.www.example.com`. While
that looks weird, it works.

If you have a simple application, that might be all you need to do. If you have
JavaScript tests, however, there is a whole new challenge to deal with.

When you run a "normal" request or feature spec, the test request is passed
through the normal routing layer of Rails, however, it isn't sent via a web
server. For a feature spec that requires JS, however, a web server is started
and the requests are sent via the server as if you were manually testing in the
browser.

By default, the web server will be listening to requests on 127.0.0.1. As we
learnt earlier, subdomains aren't supported on localhost so nothing is loaded.
Feature tests are facilitated by Capybara so you need to tell Capybara to use
a different domain. The catch is that, since you are communicating with a real
web server, you need to send the request to the port number that the server is
listening on. When you make requests to paths (rather than URLs), Capybara
handles this for you. When sending the full URL, however, you are telling
Capybara to connect via a particular port (80 if none is specified).

The following is how I've gotten around this. I found documentation on the issue
to be terribly lacking and so I can't say if it is the best solution, but I can
tell you it is _a_ solution.

Ultimately, what I wanted was a way to leave the bulk of my test code exactly as
it is but somehow tell Capybara which subdomain to run on regardless of whether
the test goes via a web server or directly to rack.

The first step is to find your Capybara configuration which is likely to be in
`spec/rails_helper.rb` and amend it to something like:
```ruby
::Capybara.configure do |config|
  config.always_include_port = true
  config.default_host = "http://easi.dev.au"
  config.server = :puma
  config.server_port = rand(50_000..59_999)
end
```

`always_include_port = true` allows us to continue sending paths and leave it
to Capybara to work out which port to send the request on.
`config.default_host` will handle any requests not on a subdomain. You may not
allow requests without a subdomain but it's a good idea to set this.
`config.server_port` technically shouldn't be needed but I haven't yet found a
way to set the `asset_host` without manually setting the port number. When you
set the `app_host`, the `always_include_port` config directs Capybara to
automatically set the port number on the request but I've found it does not set
the port number on the `asset_host`. Without it, your tests will still pass but
debugging in the browser becomes difficult because your CSS won't load.

Now that Capybaya is primed, we need a way to tell it which subdomain to use.
I've settled on adding metadata to the spec file like:
```ruby
RSpec.describe "Doing a thing", subdomain: :service do
```

If you have reason to have a spec file that tests multiple subdomains, that
metadata can be placed on a lower-level block like:
```ruby
RSpec.describe "Doing a thing" do
  describe "when on the service domain", subdomain: :service do
    it "does things"
  end

  describe "when on the support domain", subdomain: :support do
    it "does stuff"
  end
end
```

The Rspec configuration includes hooks to jump in before and after specs run and
this is where you can listen for the metadata and change the subdomain. There
are many ways to configure Rspec but I chose to keep this self-contained. My
project includes a `spec/support` directory where I created a new class which I
call at the end of my `spec/rails_helper.rb` with `Metadata::Subdomain.call`.

`spec/support/metadata/subdomain.rb`
```ruby
# frozen_string_literal: true

module Metadata
  class Subdomain
    class << self
      def call
        RSpec.configure do |config|
          before(config)
          after(config)
        end
      end

      private

      def before(config)
        config.before(:each, when_tagged_with_subdomain) do |example|
          Metadata::Subdomain.new(example).before
        end
      end

      def after(config)
        config.after(:each, when_tagged_with_subdomain) do |example|
          Metadata::Subdomain.new(example).after
        end
      end

      def when_tagged_with_subdomain
        { subdomain: ->(subdomain) { subdomain.present? } }
      end
    end

    attr_reader :example

    def initialize(example)
      @example = example
    end

    def before
      Capybara.app_host = app_host
      Capybara.asset_host = asset_host
    end

    def after
      Capybara.app_host = Capybara.default_host
      Capybara.asset_host = "#{Capybara.default_host}:#{Capybara.server_port}"
    end

    private

    def app_host
      uri = URI(Capybara.default_host)
      "#{uri.scheme}://#{subdomain}.#{uri.host}"
    end

    def subdomain
      example.metadata.fetch(:subdomain)
    end

    def asset_host
      uri = URI(Capybara.default_host)
      "#{uri.scheme}://#{subdomain}.#{uri.host}:#{Capybara.server_port}"
    end
  end
end
```

It's a lot, I know, but this complexity will rarely be seen and I found it made
the tests a lot easier to read and write.

The summary of what is going on there is that it registers callbacks to be run
before and after specs. The callbacks are only called if `subdomain` is in the
metadata and a value for the subdomain was passed.

Before the spec is run, Capybara is given a new `app_host` that includes the
subdomain. Once the test is complete, the `app_host` is set back to the default.

Hopefully, after all that, your tests should pass and you should be ready to
commit.
