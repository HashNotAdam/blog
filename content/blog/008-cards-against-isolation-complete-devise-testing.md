---
title: "Cards Against Isolation (Complete Devise Testing)"
date: 2020-09-03T20:00:00+10:00
slug: "cards-against-isolation-complete-devise-testing"
description: ""
keywords: []
draft: false
tags: [development, tutorial]
math: false
toc: false
---
In the last post, we tested authentication but we didn’t actually complete all
the Devise related testing. We really need to make sure that players can
register and reset their password if they forget it.

## Registration

We have a spec file, `spec/features/player_authentication_spec.rb`, for testing
authentication so let’s start by making a similar file for registration at
`spec/features/player_registration_spec.rb`:
```ruby
# frozen_string_literal: true

describe "Player registration", type: :feature do
end
```

The first thing we are going to want to know is whether pressing the "Sign up"
button on the sign-in page takes the player to the registration page.
```ruby
describe "Player registration", type: :feature do
  context "when a player clicks the registration link" do
    it "takes them to the registration page" do
    end
  end
end
```

I've seen people really overthink those stories. Don't! You can always modify it
later. Now is the time to feel things out. Wait until after your test is passing
to worry about the little details.

We know that this story starts on the sign-in page, so we'll need to load that
page. There will need to be an element for the registration link that needs to
be clicked. And we need a concept of a registration page to check that clicking
the link took us there.
```ruby
it "takes them to the registration page" do
  sign_in_page = Pages::Players::SignIn.new
  registration_page = Pages::Players::Registration.new

  sign_in_page.load

  sign_in_page.registration_link.click

  expect(registration_page).to be_displayed
end
```

Okay, you should know the deal by now.
1. Run the test
1. Read the error
1. Correct the error
1. Repeat

One thing to note is that we now have multiple test files. You don't need to run
all of your tests during the build process. You absolutely should run all of
your tests when you believe you have completed a feature but you'll want to get
quick responses from RSpec until that point.

You can run just one file by appending the file path to the command like:
```bash
rspec spec/features/player_registration_spec.rb
```
You can also include a line number for the test (any line number from the `it`
to the `end` will do) to only run one test in the file:
```bash
rspec spec/features/player_registration_spec.rb:5
```

I would recommend you find a shortcut that suits your workflow. I like to use
the "Rails Run Specs" extension for VS Code. Code has a built-in terminal and
"Rails Run Specs" adds shortcuts for running your tests in the terminal:
- command-shift-t : run all specs in the file
- command-l : run only the spec/s for the selected line
- command-y : re-run the last spec that was run

In the past, I've used the [Guard](https://github.com/guard/guard) gem with both
[Guard::RSpec](https://github.com/guard/guard-rspec) and
[guard-rubocop](https://github.com/yujinakayama/guard-rubocop). Guard runs in
its own terminal window and will watch you to see when you make file changes.

You designate tasks to be performed in response to file changes and those tasks
can be reasonably intelligent. For instance, the Rubocop task will only check
your changes—not the whole project—and the RSpec task can work out which spec
file relates to the changes and run just those specs. I still really like Guard
but, when you are working on a team, the team needs to agree to it whereas you
can add an extension to your editor without disrupting others.

Test error:
```
Failure/Error: registration_page = Pages::Players::Registration.new

NameError:
  uninitialized constant Pages::Players::Registration
```
We create the bare-bones registration page class at
`spec/pages/players/registration.rb`:
```ruby
# frozen_string_literal: true

module Pages
  module Players
    class Registration < SitePrism::Page
    end
  end
end
```

Test error:
```
Failure/Error: sign_in_page.registration_link.click

NoMethodError:
  undefined method `registration_link'
```

The variable, `sign_in_page`, which is an instance of `Pages::Players::SignIn`,
doesn't yet know about the registration link, so let's add it:
```ruby
element :login_button, "#new_player input[type=submit]"

element :registration_link, "FIX_ME"

element :alert, ".alert"
```
Devise doesn't make it easy for us to find this link. The HTML displayed on the
page is simply:
```html
<a href="/players/sign_up">Sign up</a>
```
so we can either find a link with the href "/players/sign_up" or where the text
reads "Sign up". Generally, I wouldn't like to hook into those attributes
because they are implementation details. We only want to test that pressing the
registration link takes us to the registration page, whether the text says
"Sign up" or "Register" or if the link is "/players/sign_up" or "/register"
really doesn’t matter. Devise does offer a way to override the default templates
but we should test that things work first and then we can refactor later. If we
were to change things now and the functionality was broken, we wouldn’t know if
we broke it in the refactor or it was never working.

The easiest attribute to find will be the path, or href:
```ruby
element :registration_link, "a[href='/players/sign_up']"
```

This link is _not_ inside the form so we omit "#new_player".

Test error:
```
Failure/Error: expect(registration_page).to be_displayed

SitePrism::NoUrlMatcherForPageError:
  SitePrism::NoUrlMatcherForPageError
```
Set the url that we found earlier in the registration page class:
```ruby
class Registration < SitePrism::Page
  set_url "/players/sign_up"
end
```

This test should be passing. Now that we know we can get to the registration
page, let's make sure we can actually register. The first step in registration
will be submitting the form and checking that a player is added to the database.

Sometimes, in the excitement of getting a green test, you can forget to test the
unhappy path. Not only can you create more than one test at a time, you can also
write out the stories without any expections and RSpec will remind you that you
have pending tests. So, let’s write out a few stories:
```ruby
context "when the registration form is filled in correctly" do
  it "creates a new, unconfirmed player" do
  end
end

context "when the email address is missing" do
  it "shows an error message"
end

context "when the email address is invalid" do
  it "shows an error message"
end

context "when the password is missing" do
  it "shows an error message"
end

context "when the password is too short" do
  it "shows an error message"
end

context "when the password confirmation is missing" do
  it "shows an error message"
end

context "when the password confirmation does not match the password" do
  it "shows an error message"
end
```

Running your tests now will show 6 pending specs. The one we will work on now
includes a do/end block so it is not counted.

Now that we've protected ourselves from ourselves, we can fill in our first
expectation:
```ruby
context "when the registration form is filled in correctly" do
  it "creates a new, unconfirmed player" do
    registration_page = Pages::Players::Registration.new
    registration_page.load

    registration_page.email_field.set("player@example.com")
    registration_page.password_field.set("Passw0rd")
    registration_page.password_confirmation_field.set("Passw0rd")

    expect { registration_page.register_button.click }.
      to change(Player, :count).by 1

    new_player = Player.find_by(email: "player@example.com")
    expect(new_player).not_to be_confirmed

    sign_in_page = Pages::Players::SignIn.new
    expect(sign_in_page).to be_displayed
  end
end
```
The first 5 lines are the same as we have seen previously. The next bit:
```ruby
expect { registration_page.register_button.click }.
  to change(Player, :count).by 1
```
is making sure that a new player is added to the database. We place
`registration_page.register_button.click` inside a block because we need to be
sure that it isn't run until RSpec is ready. Before executing that code, RSpec
needs to check how many players are in the database. It does this before and
after running the code in the block to see if the value changes.

If you wanted to know how many players are in the database, you would run
`Player.count`. Of course, if we had written `change(Player.count)`, we would
be sending the message "0" to the change method. Instead, we need to tell RSpec
what message to send so it can run it before and after.

Then we want to know that, while the player is in the database, they have not
been confirmed:
```ruby
new_player = Player.find_by(email: "player@example.com")
expect(new_player).not_to be_confirmed
```

"be_confirmed" is a dynamically called method. There isn't an RSpec called
"be_confirmed" but it will automatically grab whatever comes after "be_" and
check that method as a question. This particular line is the equivilent of:
```ruby
expect(new_player.confirmed?).to be false
```
but it reads more like a sentence in our story as "expect new player not to be
confirmed".

I was making an educated guess that "new_player" had a "confirmed?" method. It
turned out to be correct but Devise does list it in the
[documentation for confirmable](https://www.rubydoc.info/github/heartcombo/devise/master/Devise/Models/Confirmable).

And finally, we redirect back to the sign-in page.

Test error:
```
Failure/Error: registration_page.email_field.set("player@example.com")

NoMethodError:
  undefined method `email_field'
```

Devise uses the same ID for the form on the registration page as the sign-in
page, `new_player`, and it continues to use the Rails form naming conventions.
You can assume that each input element will have a name attribute of
`model[attribute]` or, in this case, `player[email]`:
```ruby
class Registration < SitePrism::Page
  set_url "/players/sign_up"

  element :email_field, "#new_player input[name='player[email]']"
end
```

Test error:
```
Failure/Error: registration_page.password_field.set("Passw0rd")

NoMethodError:
  undefined method `password_field'
```

Following the convention:
```ruby
element :email_field, "#new_player input[name='player[email]']"
element :password_field, "#new_player input[name='player[password]']"
```

Test error:
```
Failure/Error: registration_page.password_confirmation_field.set("Passw0rd")

NoMethodError:
  undefined method `password_confirmation_field'
```

Again, but over two lines else we will surpass our 80 character line limit and
the cop will be quite upset:
```ruby
element :password_confirmation_field,
        "#new_player input[name='player[password_confirmation]']"
```

Test error:
```
Failure/Error:
  expect { registration_page.register_button.click }.
    to change(Player, :count).by 1

NoMethodError:
  undefined method `register_button'
```

Fixed by:
```ruby
element :password_confirmation_field,
        "#new_player input[name='player[password_confirmation]']"
element :register_button, "#new_player input[type=submit]"
```

And we're passing already; how simple was that‽

I don't want to over-optimise but it is inevitable that every test in this file
is going to need an instance of `Pages::Players::Registration` so I'd like to
extract that into a shared variable:
```ruby
describe "Player registration", type: :feature do
  let(:registration_page) { Pages::Players::Registration.new }

  context "when a player clicks the registration link" do
```

- Run the tests to make sure nothing has broken;
- remove the two lines that say
`registration_page = Pages::Players::Registration.new`; then
- run the tests again.

Now we can fill in the bodies of the remaining tests. I know from testing the
page manually that Devise uses its own error messages for this page, not the
Rails flash messages that are used for sign-in. I'm not really sure why this
discrepancy exists. These errors are only shown when there is an issue related
to the form submission and, as such, we can simply test that an error is shown,
not what the error is. This goes back to the idea that we are not testing
implementation details—we are not interested in the specific words shown,
only that the expected reaction occurs.
```ruby
context "when the email address is missing" do
  it "shows an error message" do
    registration_page.load

    registration_page.password_field.set("Passw0rd")
    registration_page.password_confirmation_field.set("Passw0rd")
    registration_page.register_button.click

    expect(registration_page).to have_error_container
  end
end

context "when the email address is invalid" do
  it "shows an error message" do
    registration_page.load

    registration_page.email_field.set("not and email address")
    registration_page.password_field.set("Passw0rd")
    registration_page.password_confirmation_field.set("Passw0rd")
    registration_page.register_button.click

    expect(registration_page).to have_error_container
  end
end

context "when the password is missing" do
  it "shows an error message" do
    registration_page.load

    registration_page.email_field.set("player@example.com")
    registration_page.password_confirmation_field.set("Passw0rd")
    registration_page.register_button.click

    expect(registration_page).to have_error_container
  end
end

context "when the password is too short" do
  it "shows an error message" do
    registration_page.load

    registration_page.email_field.set("player@example.com")
    registration_page.password_field.set("Passw")
    registration_page.password_confirmation_field.set("Passw")
    registration_page.register_button.click

    expect(registration_page).to have_error_container
  end
end

context "when the password confirmation is missing" do
  it "shows an error message" do
    registration_page.load

    registration_page.email_field.set("player@example.com")
    registration_page.password_field.set("Passw0rd")
    registration_page.register_button.click

    expect(registration_page).to have_error_container
  end
end

context "when the password confirmation does not match the password" do
  it "shows an error message" do
    registration_page.load

    registration_page.email_field.set("player@example.com")
    registration_page.password_field.set("Passw0rd")
    registration_page.password_confirmation_field.set("This does not match")
    registration_page.register_button.click

    expect(registration_page).to have_error_container
  end
end
```

Most of this should look familar to you now apart from:
```ruby
expect(registration_page).to have_error_container
```
This is simply saying that an element called `error_container` will be somewhere
on the page. The error container is only shown when there is an error, otherwise
it doesn't exist on the page.

Test error:
```
Failure/Error: expect(registration_page).to have_error_container
  expected #<Pages::Players::Registration:0x00007f91891ac440> to respond to `has_error_container?`
```
Of course, since we haven't told SitePrism about an element called
"error_container", the test will fail. From inspecting the page, I can tell you
that the container has an ID of `error_explanation` so the element added as:
```ruby
element :register_button, "#new_player input[type=submit]"

element :error_container, "#error_explanation"
```

And, like that, we now have 8 passing tests. It can take a while to get some
momentum but, once you get those foundations in place, things can really move.

So we know that an unconfirmed user is added to the database but how do we
simulate receiving an email and clicking the confirmation link? The answer isn't
obvious; I've learnt to answer questions like these through much Googling over
the years. What I can say, however, is that the answer should always start with
a test.

Since this is the step after the existing happy path, let's keep it
in the same context:
```ruby
context "when the registration form is filled in correctly" do
  it "creates a new, unconfirmed player" do
    ...
  end

  context "when the player clicks the confirmation link in their email" do
    it "confirms their account" do
    end
  end
end
```

To test this, we need a player that has registered. We could fill in the
registration form again but we have already proven that works in another test
so we don't want to test it again. Instead, we can just create an unconfirmed
player like we did in the authentication spec:
```ruby
context "when the player clicks the confirmation link in their email" do
  let(:unconfirmed_player) do
    Player.create!(
      email: "player@example.com",
      password: "Passw0rd"
    )
  end

  it "confirms their account" do
    unconfirmed_player
  end
end
```

We can't (easily) simulate receiving an email and clicking a link but there
would surely be a way to ask Devise for that link. We know that a link can only
work if Rails has been told about it in the routes so that's a great place to
start (`rails routes`):

Prefix                  | Verb | URI Pattern                         | Controller#Action
------------------------|------|-------------------------------------|----------------------------
new_player_confirmation | GET  | /players/confirmation/new(.:format) | devise/confirmations#new
player_confirmation     | GET  | /players/confirmation(.:format)     | devise/confirmations#show
_                       | POST | /players/confirmation(.:format)     | devise/confirmations#create

Those are all the routes relating to confirming a player. Since the confirmation
occurs after a link is clicked in an email, we can be reasonably certain that
the verb will be "GET". By default, links are always going to be GET requests.
In a website, it is not only possible to change this but Rails does this to
allow things like deleting records but you can safely assume that a link in an
email is going to be a GET whereas POST is going to be the submission of a form.

That leaves us with `new_player_confirmation`, and `player_confirmation`. The
`new` action should show you a page for creating something so we can expect that
it won't have any knowledge of our specific confirmation. That leaves us with
the `show` action. It certainly isn't usual for a `show` action to perform a
side-effect but it would need to relate to a specifc confirmation and would
generally take some kind of identifier. If this was a Rails controller backed
by an object in the database, the show links would be like
`model_name/id_of_thing`.

Because I know that there needs to be an identifier and I can't see one in the
URI pattern, I jumped into the Devise source code to move forward. I fully
appreciate that this isn't the simpliest path for people who are not yet
comfortable navigating Ruby projects. Another way to solve this problem would be
to complete the registration form in the browser and check the link in the email
but we haven’t configured the ability to send emails yet.

The results from the routes said that the controller is `devise/confirmations`.
Since Devise is specifically for Rails, I had an expectation that it would
follow Rails conventions and so I expected to find a ConfirmationsController
inside `app/controllers` just like how my own controllers are defined. When I
opened [app/controllers](https://github.com/heartcombo/devise/tree/v4.7.2/app/controllers)
on GitHub I could see a `devise` directory and inside
[that directory](https://github.com/heartcombo/devise/tree/v4.7.2/app/controllers/devise)
was [confirmations_controller](https://github.com/heartcombo/devise/blob/v4.7.2/app/controllers/devise/confirmations_controller.rb).
That file has a [show method](https://github.com/heartcombo/devise/blob/16f27b3074c544c868335898c207bf6d2152c929/app/controllers/devise/confirmations_controller.rb#L21)
which has this helpful comment above:
```ruby
# GET /resource/confirmation?confirmation_token=abcdef
```

We now know that we need to send a request to the `player_confirmation_path`
and there needs to be a `confirmation_token` in the URL parameters. The URL
parameter can be passed to the path like
`player_confirmation_path(confirmation_token: some_token)`.

So, where do we get that token? Thankfully, Devise puts it on the player with
exactly that name (`unconfirmed_player.confirmation_token`).

Putting it all together:
```ruby
it "confirms their account" do
  confirmation_page = Pages::Players::Confirmation.new
  token = unconfirmed_player.confirmation_token

  confirmation_page.load(token: token)
end
```
Because we are calling `unconfirmed_player` when we load the page, we no longer
need to call it at the start of the test.

To be certain that the player is being transitioned from unconfirmed to
confirmed, we will run our expectation in a block like we did in
`it "creates a new, unconfirmed player" do`:
```ruby
it "confirms their account" do
  confirmation_page = Pages::Players::Confirmation.new
  token = unconfirmed_player.confirmation_token

  expect { confirmation_page.load(token: token) }.
    to change { unconfirmed_player.reload.confirmed? }.to true
end
```

Previously when we used the `change` method we passed the parameters in
parentheses (`change(Player, :count)`) but now we are using a block. This is
because the first option only supports one method call and we are chaining two
methods. But why are we doing that? We create the player and store it to a
variable. When Devise confirms the player, it puts a date into the
`confirmed_at` column in the database. Our `unconfirmed_player` variable doesn't
know that happened. `unconfirmed_player` is essentially a cache of what the
database looked like at the point we set the variable. When we call`reload`,
Active Record goes back to the database and updates the cache.

Test error:
```
Failure/Error: confirmation_page = Pages::Players::Confirmation.new

NameError:
  uninitialized constant Pages::Players::Confirmation
```

Create the page class, `spec/pages/players/confirmation.rb`:
```ruby
# frozen_string_literal: true

module Pages
  module Players
    class Confirmation < SitePrism::Page
    end
  end
end
```

Test error:
```
Failure/Error:
  expect { confirmation_page.load(token: token) }.
    to change { unconfirmed_player.reload.confirmed? }.to true

SitePrism::NoUrlForPageError:
  SitePrism::NoUrlForPageError
```

Add the URL to the page class:
```ruby
class Confirmation < SitePrism::Page
  set_url "/players/confirmation?confirmation_token={token}"
end
```
That "{token}" is how SitePrism allows us to pass parameters into the URL and
is what makes `confirmation_page.load(token: token)` work.

And with that test complete, we have completed testing registration so we should
commit:
```bash
rubocop
rspec
git add --intent-to-add spec
git add --patch
git commit
```

> Add tests for player registration

## Forgot password

Now that we have completed registration, this should be a breeze. We have all
the knowledge required to fly though it, it's just a matter of applying that
knowledge to a different page.

As with registration, we start by creating a spec file at
`spec/features/player_forgotten_password_spec.rb`:
```ruby
# frozen_string_literal: true

describe "Player forgotten password", type: :feature do
end
```

The registration spec began:
```ruby
let(:registration_page) { Pages::Players::Registration.new }

context "when a player clicks the registration link" do
  it "takes them to the registration page" do
    sign_in_page = Pages::Players::SignIn.new

    sign_in_page.load

    sign_in_page.registration_link.click

    expect(registration_page).to be_displayed
  end
end
```
so let's reword that for forgotten password:
```ruby
let(:forgotten_password_page) { Pages::Players::ForgottenPassword.new }

context "when a player clicks the forgotten password link" do
  it "takes them to the forgotten password page" do
    sign_in_page = Pages::Players::SignIn.new

    sign_in_page.load

    sign_in_page.forgotten_password_link.click

    expect(forgotten_password_page).to be_displayed
  end
end
```

If you are calling RSpec manually, now you will want to call:
```bash
rspec spec/features/player_forgotten_password_spec.rb
```

Test error:
```
Failure/Error: sign_in_page.forgotten_password_link.click

NoMethodError:
  undefined method `forgotten_password_link'
```

Before we can create the `forgotten_password_link` element, we need to work out
what the path is. Running `rails routes` shows us:

Prefix               | Verb  | URI Pattern                      | Controller#Action
---------------------|-------|----------------------------------|------------------------
new_player_password  | GET   | /players/password/new(.:format)  | devise/passwords#new
edit_player_password | GET   | /players/password/edit(.:format) | devise/passwords#edit
player_password      | PATCH | /players/password(.:format)      | devise/passwords#update
_                    | PUT   | /players/password(.:format)      | devise/passwords#update
_                    | POST  | /players/password(.:format)      | devise/passwords#create

`passwords#new` is the most likely candidate here since we want a page that will
allow us to request a new password. Let's look at the other options, though.

`edit` and `update` do seem like they could make sense but, apart from the fact
we can look at the URL in the browser which shows `/players/password/new`, we
would expect that the page we are looking for will let us request an email
with a link we can click to get to a page to update our password. Since the
final action is updating the password and you update a record from an edit page,
it seems most likely that the page we are looking for creates an edit link.

As with the confirmation link, we can exclude the `create` action since POST
actions should create records and we only want to load a page.

Add the `forgotten_password_link` element to `Pages::Players::SignIn`:
```ruby
element :registration_link, "a[href='/players/sign_up']"
element :forgotten_password_link, "a[href='/players/password/new']"
```

Test error:
```
Failure/Error: let(:forgotten_password_page) { Pages::Players::ForgottenPassword.new }

NameError:
  uninitialized constant Pages::Players::ForgottenPassword
```

Create `spec/pages/players/forgotten_password.rb`:
```ruby
# frozen_string_literal: true

module Pages
  module Players
    class ForgottenPassword < SitePrism::Page
    end
  end
end
```

Test error:
```
Failure/Error: expect(forgotten_password_page).to be_displayed

SitePrism::NoUrlMatcherForPageError:
  SitePrism::NoUrlMatcherForPageError
```

Add the URL:
```ruby
class ForgottenPassword < SitePrism::Page
  set_url "/players/password/new"
end
```

That should be all to get the test passing.

The forgotten password page only has one field, email address, and a submit
button. Following the examples from registration, our stories are:
```ruby
context "when the forgotten password form is filled in correctly" do
  it "sets reset_password_token on the player"

  context "when the player clicks the password reset link in their email" do
    it "allows them to change their email address"
  end
end

context "when the email address is missing" do
  it "shows an error message"
end

context "when the email address does not match a player" do
  it "shows an error message"
end
```

In the first test, I knew that Devise would have to store a token for the
forgotten password functionality to work (as with confirmation) so I checked
`config/schema.rb` looking for something that made sense.

The first test will be very similar to
`it "creates a new, unconfirmed player"` in registration. For this test to
work, however, we are going to need to have a confirmed player in the database
(you can't reset the password for a player that doesn't exist). A confirmed
player is something we are going to need regularly throughout our testing. The
majority of the application simply won't work without being signed in as a
confirmed player. Given this, it is going to make our lives simplier if there is
a player variable that we can call in all of our tests. There are multiple ways
to do this but I'm a fan of the
[factory_bot gem](https://github.com/thoughtbot/factory_bot_rails).

factory_bot gets added to your development and test group in the Gemfile:
```ruby
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem "byebug", platforms: %i[mri mingw x64_mingw]
  gem "factory_bot_rails", "~> 6.1.0"
  gem "rspec-rails", "~> 4.0.1"
end
```

Make sure you install:
```bash
bundle
```

Now factory_bot needs to be loaded into RSpec. The factory_bot documentation
recommends creating a `spec/support` directory and then adding the file
`spec/support/factory_bot.rb`:
```ruby
# frozen_string_literal: true

RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
end
```
Then this file needs to be included in `spec/rails_helper.rb`. Previously we
included all the files in `spec/pages` by adding:
```ruby
Dir[Rails.root.join("spec/pages/**/*.rb")].sort.each { |f| require f }
```
Directly above this line should be:
```ruby
# Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }
```
Remove the hash and update to read:
```ruby
Dir[Rails.root.join("spec/support/**/*.rb")].sort.each { |f| require f }
```

As always, commit this as a separate story arc:
```
git add --intent-to-add spec/support
git add --patch Gemfile* spec/rails_helper.rb spec/support
git commit
```

> Install factory_bot_rails

To create your first factory, create the directory `spec/factories` and then
create the file `spec/factories/player.rb`.
```ruby
# frozen_string_literal: true

FactoryBot.define do
  factory :player do
  end
end
```

By default, factory_bot will assume that the name of your factory is also the
name of the model. In our tests, if we write `create(:player)`, factory_bot will
create a new player in the database. `build(:player)` will give you a new player
but that player will not yet be stored in the database. Inside the `factory`
block, we can set the defaults we want to use:
```ruby
FactoryBot.define do
  factory :player do
    email { "player@example.com" }
    password { "Passw0rd" }
    confirmed_at { Time.zone.now }
  end
end
```
By setting the `confirmed_at` date, the player will be confirmed when we create
it.

Now we can create the test:
```ruby
context "when the forgotten password form is filled in correctly" do
  let(:player) { create(:player) }

  it "sets reset_password_token on the player" do
    forgotten_password_page.load

    forgotten_password_page.email_field.set(player.email)

    expect { forgotten_password_page.reset_password_button.click }.
      to change { player.reload.reset_password_token }
  end

  context "when the player clicks the password reset link in their email" do
    it "allows them to change their email address"
  end
end
```

Test error:
```
Failure/Error: forgotten_password_page.email_field.set(player.email)

NoMethodError:
  undefined method `email_field'
```

Add `email_field` to `Pages::Players::ForgottenPassword` _exactly_ the same as
in `Registration`:
```ruby
class ForgottenPassword < SitePrism::Page
  set_url "/players/password/new"

  element :email_field, "#new_player input[name='player[email]']"
end
```

Test error:
```
Failure/Error:
  expect { forgotten_password_page.reset_password_button.click }.
    to change { player.reload.reset_password_token }

NoMethodError:
  undefined method `reset_password_button'
```

Add the `reset_password_button` element:
```ruby
element :email_field, "#new_player input[name='player[email]']"
element :reset_password_button, "#new_player input[type=submit]"
```

Assuming the test is now green, we can move onto the unhappy paths. These are
nothing more than modified versions of the registration tests:
```ruby
context "when the email address is missing" do
  it "shows an error message" do
    forgotten_password_page.load

    forgotten_password_page.reset_password_button.click

    expect(forgotten_password_page).to have_error_container
  end
end

context "when the email address does not match a player" do
  it "shows an error message" do
    forgotten_password_page.load

    forgotten_password_page.email_field.set("invalid@example.com")
    forgotten_password_page.reset_password_button.click

    expect(forgotten_password_page).to have_error_container
  end
end
```

Test error:
```
Failure/Error: expect(forgotten_password_page).to have_error_container

NoMethodError:
  undefined method `has_error_container?'
```

Add `error_container`:
```ruby
element :reset_password_button, "#new_player input[type=submit]"

element :error_container, "#error_explanation"
```

The final test for this section is:
```ruby
context "when the player clicks the password reset link in their email" do
  it "allows them to change their email address" do
    token = player.send_reset_password_instructions

    reset_password_page = Pages::Players::ResetPassword.new
    reset_password_page.load(token: token)

    reset_password_page.password_field.set("Password_2")
    reset_password_page.password_confirmation_field.set("Password_2")

    expect { reset_password_page.reset_password_button.click }.
      to change { player.reload.encrypted_password }
  end
end
```

The method `send_reset_password_instructions` will create the
`reset_password_token` on the player. Devise encrypts the token and so the token
stored to `reset_password_token` is different from the one used in the URL. I
only found this out from many failures over the years and a lot of Googling. I
found `send_reset_password_instructions` in the
[Devise test files](https://github.com/heartcombo/devise/blob/16f27b3074c544c868335898c207bf6d2152c929/test/mailers/reset_password_instructions_test.rb#L20).

Test error:
```
Failure/Error: reset_password_page = Pages::Players::ResetPassword.new

NameError:
  uninitialized constant Pages::Players::ResetPassword
```

Create `spec/pages/players/reset_password.rb`:
```ruby
# frozen_string_literal: true

module Pages
  module Players
    class ResetPassword < SitePrism::Page
    end
  end
end
```

Test error:
```
Failure/Error: reset_password_page.load(token: "token")

SitePrism::NoUrlForPageError:
  SitePrism::NoUrlForPageError
```

Based on the table of routes, we assumed that the `edit` action would be the
most likely place for the link to take us. Checking the
[controller in the Devise code](https://github.com/heartcombo/devise/blob/16f27b3074c544c868335898c207bf6d2152c929/app/controllers/devise/passwords_controller.rb#L25),
we see the comment:
```ruby
# GET /resource/password/edit?reset_password_token=abcdef
```
which leads us to conclude that the url should be set as:
```ruby
class ResetPassword < SitePrism::Page
  set_url "/players/password/edit?reset_password_token={token}"
end
```

Test error:
```
Failure/Error: reset_password_page.password_field.set("Password_2")

NoMethodError:
  undefined method `password_field'
```

Add the element to the page class:
```ruby
set_url "/players/password/edit?reset_password_token={token}"

element :password_field, "#new_player input[name='player[password]']"
```

Test error:
```
Failure/Error: reset_password_page.password_confirmation_field.set("Password_2")

NoMethodError:
  undefined method `password_confirmation_field'
```

Now you can add the confirmation field:
```ruby
element :password_field, "#new_player input[name='player[password]']"
element :password_confirmation_field,
        "#new_player input[name='player[password_confirmation]']"
```

Test error:
```
Failure/Error:
  expect { reset_password_page.reset_password_button.click }.
    to change { player.reload.encrypted_password }

NoMethodError:
  undefined method `reset_password_button'
```

And, finally, the submit button:
```ruby
element :password_field, "#new_player input[name='player[password]']"
element :password_confirmation_field,
        "#new_player input[name='player[password_confirmation]']"
element :reset_password_button, "#new_player input[type=submit]"
```

You should now be green and can commit again:
```bash
rubocop
rspec
git add --intent-to-add spec
git add --patch
git commit
git push
```

> Add tests for player forgotten password

That's all the testing we are going to do for now. My code at this point is
[available on GitHub](https://github.com/HashNotAdam/cards-against-isolation/commit/8cd8c9e230ae4fac0563cfe28e258c58177a3c03).

[Next week](/blog/cards-against-isolation-base-styles) we'll hang up the testing
hat and start making pretty things.
