---
title: "Cards Against Isolation (Creating A Game)"
date: 2020-09-15T10:37:11+10:00
slug: "cards-against-isolation-creating-a-game"
description: ""
keywords: []
draft: false
tags: [development, tutorial]
math: false
toc: false
---
I'm quite excited about this post. Today we'll be installing
[ViewComponent](https://github.com/github/view_component) and creating our
first components. ViewComponent is still reasonably new and so I haven't had an
opportunity to try it yet.

The real value that I see in a ViewComponent compared to using Rails partials
for reusability is the testability of component. Testing your views can be a
slow process. We have created some feature specs to ensure Devise is working
but that requires the test suite to load a page and then parse it looking for
particular elements and strings of content. While feature tests are a powerful
way to test how various pieces of your application interact, they are
unquestionably slower and you want to only produce the minimum number needed to
provide broad confidence. On the other hand, components support unit tests and
so you can create lots of small tests of isolated features to ensure your
components work in a range of scenarios without too much concern for test speed.
At least, that's the theory.

The tasks for today will be to create:
- a link on the dashboard for creating a new game;
- a new game form; and
- the beginnings of a game page.

## Testing

Before we get to play, we need to put the work in and create some tests. We may
not have a game model yet but that doesn't mean we can't document that we expect
a game to be created. The tests will fail initially but they will be our guide
as we build.

Start by creating a new spec file, `spec/features/create_a_game_spec.rb`:
```ruby
# frozen_string_literal: true

describe "Create a game", type: :feature do
end
```

There are three stories I want to test:
- when a player clicks on a link on the dashboard, they're taken to the new game
form;
- if they attempt to create a game with a name that already exists, an error is
thrown; and
- when they submit the new game form:
  - a game is created;
  - the player is the game owner;
  - the player has joined the game; and
  - the player was redirected to the game page.

That third story has a lot more going on. One of the downsides to feature tests
is, because they make a page request, they are slow and so you want to pack the
related expectations together. It’s a bit of a balancing act because, while you
could test all stories together, they are independent and so a failure could
exist in many, unrelated places. When we do more unit testing, you’ll notice
that there is a desire to make many more tests for each expectation. Unit tests
should be really fast and so you can afford to make lots of tests that stress
small sections of your app.

```ruby
describe "Create a game", type: :feature do
  context "when the new game link on the dashboard is pressed" do
    it "redirects to the new game page"
  end

  context "when the new game form is submitted correctly" do
    it "creates the game and redirects to the game page"
  end

  context "when the new game name matches an existing game" do
    it "shows an error message"
  end
end
```

You could decide to either fill those tests out one at a time, knowing that
Rspec will remind you that you have pending tests, or you could fill them all
out now. I’m not sure it makes a lot of difference. If you find writing the
tests hard, you might appreciate jumping between tests and application code. In
this particular case, I’d rather get the tests out of the way first and then
move onto features. Just know that, if my way wouldn’t work for you, that’s
completely fine.

First up, we are going to need a player who is logged in:
```ruby
describe "Create a game", type: :feature do
  let(:player) { create(:player) }

  before { sign_in player }
```

`sign_in` is a helper method that does what it says on the tin. Rather than
going to the sign-in form, filling in the player details, submitting the form,
and then going to the page you want, Devise can just log the player in for you.
Anything placed inside the `before` block will be run before every test so a
player will be logged in before each test.

```ruby
context "when the new game link on the dashboard is pressed" do
  it "redirects to the new game page" do
    dashboard = Pages::Dashboards::Index.new
    dashboard.load

    dashboard.new_game_link.click

    new_game_page = Pages::Games::New.new
    expect(new_game_page).to be_displayed
  end
end
```

`Pages::Dashboards::Index` is the class we created in a previous post. Because
we are using SitePrism, we don't need to think about the details of loading that
page anymore, we know that is correct since our other tests are passing. We
will, however, need to tell the class about the `new_game_link` but that
doesn't happen until our test complains about it.

`Pages::Games::New` is a class we haven't yet created but the name follows the
convention we have been using.

```ruby
context "when the new game form is submitted correctly" do
  it "creates the game and redirects to the game page" do
    new_game_page = Pages::Games::New.new
    new_game_page.load

    new_game_page.name_field.set("Isogame")
    new_game_page.create_button.click

    new_game = Game.find_by(name: "Isogame")
    expect(new_game).to have_attributes(
      name: "Isogame",
      owner_id: player.id
    )
    expect(new_game.players).to include player

    show_game_page = Pages::Games::Show.new
    expect(show_game_page).to be_displayed(id: new_game.id)
    expect(show_game_page).to have_content("Isogame")
  end
end
```

Okay, this is the big one, so let's break it down.

```ruby
new_game_page = Pages::Games::New.new
new_game_page.load

new_game_page.name_field.set("Isogame")
new_game_page.create_button.click
```
While the new game form doesn’t exist yet, we know that there can be multiple
games being played and so there will need to be some way to tell them apart, a
name. Since it is a form, it is obvious that we will need a button to submit the
form.

```ruby
new_game = Game.find_by(name: "Isogame")
expect(new_game).to have_attributes(
  name: "Isogame",
  owner_id: player.id
)
expect(new_game.players).to include player
```

Once again, we don’t have a game model yet but there will need to be one, it’s
the only way for it to be stored in the database.

`owner_id` is referring to an association. I am expecting that I will want
to create a relationship between the game and the person who created it. My
thought is that it will be up to the owner to start the game or maybe they can
stop the game early.

Similar to how `owner_id` is an association to one player, `players` will be an
association to many players who are participating in the game. It wouldn't be
strictly necessary to add the owner to the player list since you could assume
that the owner is in the game. I've historically found, however, that it is a
_lot_ nicer to simply call `players` to get all players than to do something
wacky like:
```ruby
players = [new_game.owner]
players.concat(new_game.players)
```

Finally:
```ruby
context "when the new game name matches an existing game" do
  before do
    Game.create!(
      name: "Isogame",
      owner: player
    )
  end

  it "shows an error message" do
    new_game_page = Pages::Games::New.new
    new_game_page.load

    new_game_page.name_field.set("Isogame")
    new_game_page.create_button.click

    expect(new_game_page).to have_error_message
  end
end
```

Since this test checks what happens if you try to make a duplicate game, the
first thing we need to do is create the game that will be duplicated. I do this
in a `before` block because, for me, it’s some logical state that needs to exist
before the test can be run. There will be others who might want all the setup to
be in each test so it’s obvious what state belongs to what test. I can
appreciate that argument, but I find the test too crowded that way. I do,
however, sympathise with the idea that setting your state at the top of a large
test file means it’s hard to jump between the test and the setup. Here we are
creating the game right next to the test that is using it and so everything is
close together but my brain prefers the logical groupings.

To be perfectly honest with you, it has been so long since I’ve built a Rails
form that I forgot how errors are shown. After staring at the screen for a
moment, I realised they must be posted to the flash messages we’ve look at in
the Devise testing. I obviously didn’t need to tell you that, but I want to keep
banging the it-doesn’t-need-to-be-completely-correct-initially drum. I didn’t
need to know how the error message was going to be shown to write a test that
says there will be one. Given we’ve tested flash messages in another spec, it’s
also possible that this code will change; it doesn’t matter, I’m just planning
out my expectations.

## Setting up the test helpers

As we’ve seen in the past, we progress by running the test suite and fixing the
errors one at a time.

```bash
rspec spec/features/create_a_game_spec.rb
```

Test error:
```
Failure/Error: before { sign_in player }

NoMethodError:
  undefined method `sign_in'
```

Earlier I mentioned that `sign_in` is a helper method supplied by Devise (and I
wasn't lying). The only thing is that it isn't available by default, we need
to include the appropriate helper file into Rspec as explained by the
[Devise documentation](https://github.com/heartcombo/devise/blob/v4.7.2/README.md#integration-tests).
This is actually quite simple and extremely similar to how we added the helpers
for FactoryBot. Create a new file, `spec/support/devise.rb`:
```ruby
# frozen_string_literal: true

RSpec.configure do |config|
  config.include Devise::Test::IntegrationHelpers, type: :feature
end
```

Run your tests again to make sure you get a different error and then commit this
change by itself:
```bash
rubocop
git add --intent-to-add spec/support/devise.rb
git add --patch
git commit
```

> Include Devise test helpers in feature specs

## Add a button to the dashboard

After running the test file, you should now have the error:
```
Failure/Error: dashboard.new_game_link.click

NoMethodError:
  undefined method `new_game_link'
# ./spec/features/create_a_game_spec.rb:13
```

The important takeaway from this error is it is saying that there is no method
called `new_game_link` and it shows that this happened when calling
`dashboard.new_game_link.click`. So, first we answer the question, "what is
`dashboard`?"

The last line of the error tells us where to look:
```ruby
13      dashboard.new_game_link.click
```

Our first test creates a new instance of `Pages::Dashboards::Index.new` and
assigns it to the variable, `dashboard`. Therefore, the error tells us that
`Pages::Dashboards::Index` does not have something called `new_game_link`.

Modify `spec/pages/dashboards/index.rb`:
```ruby
module Pages
  module Dashboards
    class Index < SitePrism::Page
      set_url "/dashboards"

      element :new_game_link, "a.new_game"
    end
  end
end
```

So, I've decided that the new game link will have a class of "new_game". Links
don't automatically get a class, this is something we’ll add to make it easier
to find in our tests.

Test error:
```
Failure/Error: dashboard.new_game_link.click

Capybara::ElementNotFound:
  Unable to find css "a.new_game"
```

We've told the test suite that were will be a link but we haven't actually
created it yet. We are keeping our Pages classes naming conventions in line with
Rails naming conventions so the view for `Pages::Dashboards::Index` is
`app/views/dashboards/index.html.erb`.

We left that file in a super boring state of affairs—nothing more than the
word "Dashboard". We used that text just so we could be certain we were looking
at the dashboard and not a white (or broken) screen.

We'll leave the "Dashboard" label there for now and add the new game link:
```erb
Dashboard
<%= link_to "Create a game", new_game_path, class: "new-game" %>
```

Test error:
```
Failure/Error: <%= link_to "Create a game", new_game_path, class: "new-game" %>

ActionView::Template::Error:
  undefined local variable or method `new_game_path'
```

Hopefully you recall that those `[something]_path` links come from a Rails
helper that hooks into the routes. We haven't actually created a route for games
so Rails can't find it. It's a shame that Rails doesn't see the `_path` suffix
and give you an error like "it looks like you are trying to get the path for
a route that doesn't exist, please define the route in config/routes.rb".

Anyway, that's what we're going to do:
```ruby
resources :dashboards, only: :index
resources :games, only: :new
```

We are also going to need routes for `create` and `show` but, you know the
rules by now, we only do as much as we need to for the test to pass. If more
needs to be done, we do it when the test tells us to. This ensures we haven't
missed a test. If you can't write code until the test tells you, the code must
be tested.

Test error:
```
Failure/Error: dashboard.new_game_link.click

ActionController::RoutingError:
  uninitialized constant GamesController
```

Each route maps to a controller in `app/controllers` so we need to create a new
controller at `app/controllers/games_controller.rb`:
```ruby
# frozen_string_literal: true

class GamesController < ApplicationController
end
```

Test error:
```
Failure/Error: dashboard.new_game_link.click

AbstractController::ActionNotFound:
  The action 'new' could not be found for GamesController
```

Every action in your routes needs to have a method in the controller. In this
case, we want the `new` action for games, so we need a `new` method in the
`GamesController`:
```ruby
class GamesController < ApplicationController
  def new; end
end
```

## Create a template for the new game page

Test error:
```
Failure/Error: dashboard.new_game_link.click

ActionController::MissingExactTemplate:
  GamesController#new is missing a template for request formats: text/html
```

This tells us that the last step was successful. We created and clicked on a
button and now the test is being directed to the new game page, but we haven’t
created a template for that page. The convention we followed above for editing
the `Dashboards::Index` page was `app/views/dashboards/index.html.erb`. Since
we want to edit the new game page, we need to create
`app/views/games/new.html.erb` with some extremely exciting content:
```html
<h1>Create a new game</h1>
```

Test error:
```
Failure/Error: new_game_page = Pages::Games::New.new

NameError:
  uninitialized constant Pages::Games
```

The last past of this test was to check that the new game page has been loaded.
This requires asking the `Pages::Games::New` if it is displayed. Create the
directory `spec/pages/games` and then create the class at
`spec/pages/games/new.rb`:
```ruby
# frozen_string_literal: true

module Pages
  module Games
    class New < SitePrism::Page
    end
  end
end
```

Test error:
```
Failure/Error: expect(new_game_page).to be_displayed

SitePrism::NoUrlMatcherForPageError
```

We need to set the page URL in that class:
```ruby
class New < SitePrism::Page
  set_url "/games/new"
end
```

That URL follows the routing conventions previously discussed and is also the
URL you would see if you looked at the dashboard link in your browser.

This actually completes the test so you'll get a single green dot.

Moving onto the next failing test:
```
Failure/Error: new_game_page.name_field.set("Isogame")

NoMethodError:
  undefined method `name_field'
```

Before we even create the form, we can know the CSS selectors for the fields
because they will follow the same Rails conventions we used when working on the
Devise tests.

```ruby
class New < SitePrism::Page
  set_url "/games/new"

  element :name_field, ".new_game input[name='game[name]']"
end
```

Test error:
```
Failure/Error: new_game_page.name_field.set("Isogame")

Capybara::ElementNotFound:
  Unable to find css "input[name='game[name]']"
```

Time to create the form in our template, `app/views/games/new.html.erb`. This is
the first form we have had to create ourselves and, while I prefer the idea of
making only 1 change per test error, we have no choice but to make 2 changes
here.

While we decide which fields will be in a form, Rails does an enormous amount of
the heavy lifting. To do that, however, we need to tell it what the form is for.
Since we want a form to us to create a new game, we need to pass a new game into
the form. The reason this is a 2 part change is that you never put logic into
your view. If we are going to create a new game, it needs to be done elsewhere
and passed into the view.

In this case, we can create the game in the controller and store it to an
instance variable. All instance variables defined in your controller will be
automatically made available for use in your view.

Let's start there; open `app/controllers/games_controller.rb` and modify the
`new` method:
```ruby
def new
  @game = Game.new
end
```

That is all that will be needed in the controller. That `@game` can now be
called from our view. Rails knows the difference between a new and an existing
game and will automatically handle setting up the form to create a new record
rather than editing existing record.

Moving onto `app/views/games/new.html.erb`:
```erb
<h1>Create a new game</h1>

<%= form_with model: @game, class: "new_game" do |f| %>
  <%= f.text_field :name %>
  <%= f.submit "Create" %>
<% end %>
```

It is really that simple, Rails does basically everything. I've added the
"new_game" class just to make it a little easier to find the form on the page in
our tests. While Capybara is really good at finding elements on the page, using
a specific class on the form just reduces any potential for issues in the
future if another form is added to the page, particularly when you consider that
there are no identifying characteristics to submit buttons.

So let's run the tests again:
```
Failure/Error: @game = Game.new

NameError:
  uninitialized constant GamesController::Game
```

We've asked to create a new instance of the `Game` class but that class doesn't
yet exist. Since we store games in the database, like players, the `Game` class
needs to extend `ApplicationRecord`. Create it at `app/models/game.rb`:
```ruby
# frozen_string_literal: true

class Game < ApplicationRecord
  validates :name, presence: true
  validates :name, uniqueness: true
end
```

The `validates` lines tell Rails to run checks on records before saving them.
`presence` means "make sure this field has a value" and `uniqueness` checks for
duplicate records.

Test error:
```
Failure/Error: @game = Game.new

ActiveRecord::StatementInvalid:
  PG::UndefinedTable: ERROR:  relation "games" does not exist
  LINE 8:  WHERE a.attrelid = '"games"'::regclass
```

Since games are stored in the database, we need to create a table for them. To
do that, we start by creating a database migration:
```bash
rails g migration CreateGames
```
which creates a file like `db/migrate/[date]_create_games.rb`.

Add the `# frozen_string_literal: true` line to the top of that file then edit
it accordingly:
```ruby
def change
  create_table :games do |t|
    t.string :name, null: false, index: { unique: true }
    t.references :owner, null: false, foreign_key: { to_table: :players }
    t.timestamps
  end

  add_foreign_key :games, :players, column: :owner_id
end
```

We are creating a unique database index on the `name` column which will make it
impossible to create duplicate records. We will also want to perform this
validation in the Rails model, however, that validation is subject to race
conditions.

Before creating a new record, Rails will ask the database if another record with
the same name exists. After the database responds "no", Rails will send the
request to create the record. The issue occurs when two people are trying to
create a record at the same time:
- Person 1 asks to create game "Isogame"
- Person 2 asks to create game "Isogame"
- Rails (person 1), asks the database if there is already a game called
"Isogame"
- The database responses to Rails (person 1) with "no"
- Rails (person 2), asks the database if there is already a game called
"Isogame"
- The database responses to Rails (person 2) with "no"
- Rails (person 1), asks the database to create the new game
- The new game is created
- Rails (person 2), asks the database to create the new game
- The database refuses because person 1 already created that game

A unique index is your final line of defense.

When we reference owner, we are telling Active Record to create a column called
`owner_id` and place an index on it. It is most common for this reference to
match the name of another model. In this case, we are storing the ID of the
player that created the game. Given this, it would be easier to reference
`:player` but this would be confusing. If you looked at a game model and saw
that it referenced one player, you might think this is a game for one. By
calling the field “owner”, we are giving more context about this relationship
but we will need to explain to Active Record what the relationship is between
this field and our application model.

While Active Record has a convention that database columns ending in `_id` are
associations to other tables, that convention doesn’t exist in the database.
Adding a foreign key constraint explains this relationship to the database and
allows it to check the data being stored. When storing a new game, the database
will make sure that any number placed in the `owner_id` column matches the ID of
a player in the players table. It will also ensure that you do no delete a
player if they are associated with any games.

Run this migration file with:
```bash
rails db:migrate
```

Since creating a model is a side task, I'd like to create a new commit:
```ruby
rubocop db/migrate app/models/game.rb
git add --intent-to-add app/models/game.rb db/migrate
git add --patch app/models/game.rb db/migrate db/schema.rb
git commit
```

> Create Game model

## Store the new game

Now, when we run the tests again:
```
Failure/Error: <%= form_with model: @game do |f| %>

ActionView::Template::Error:
  undefined method `games_path'
```

At this point, I want to acknowledge that some of these errors aren’t obvious;
there are some things that just require practice and experience. Here, the
problem is that Rails has worked out that we want to create a new game. When you
submit the form for a new record, it is sent to the `create` action for that
controller. The `create` URL and the `index` URL are the same; the difference is
the HTTP verb. The `index` path for games would be a `GET` request to `/games`
whereas the `create` path would be a `POST` request to `/games`.

We have created a route for the `new` action, but not the `create` action.
Update `config/routes.rb`:
```ruby
resources :games, only: %i[new create]
```

Test error:
```
Failure/Error: new_game_page.create_button.click

NoMethodError:
  undefined method `create_button
```

Moving back to the Pages class, `spec/pages/games/new.rb`:
```ruby
element :name_field, ".new_game input[name='game[name]']"
element :create_button, ".new_game input[type=submit]"
```

Test error:
```
Failure/Error: new_game_page.create_button.click

AbstractController::ActionNotFound:
  The action 'create' could not be found for GamesController
```

This error tells us that the form submission is now working and we now need to
write the logic for creating the game.

Jump back into the `GamesController` (`app/controllers/games_controller.rb`) and
create an empty `create` method:
```
def new
  @game = Game.new
end

def create
end
```

Test error:
```
Failure/Error:
  expect(new_game).to have_attributes(
    name: "Isogame",
    owner_id: player.id
  )

  expected nil to respond to :name, :owner_id with 0 arguments
```

These errors can be confusing when you first see them, but they are not only
extremely common but, once you understand them, they give you a very clear
indication of where your problem lies.

In saying that it "expected nil to respond to" messages, it is telling you that
the variable you expected to be some kind of object (a game in this case) is
actually `nil`.

In debugging this, the first question to ask yourself is "why did I expect
`new_game` to be a game?"

We are assigning `new_game` in the line:
```ruby
new_game = Game.find_by(name: "Isogame")
```

If we are asking Rails to get the game out of the database and instead it gave
us `nil`, then either the game isn't in the database or it was stored with a
different name. Here, of course, we haven't actually written any code to store
the game; our `create` method is still empty.

At part of our process of taking the smallest steps possible per iteration, we
want to create the game using hard-coded values:
```ruby
def create
  Game.create!(
    name: "Isogame",
    owner: current_player
  )
end
```

`current_player` is a helper method provided by Devise. It provides you a method
called `current_[model_name]` that returns the logged in resource. Since our
Devise "resource" is a player, `current_player` is whichever player requested
the creation of a new game.

Clearly we are not going to want to keep "Isogame" in there but first we focus
on getting the test passing and then we can replace "Isogame" with a dynamic
value coming from the form, knowing that the test will tell us when we get it
right.

Test error:
```
Failure/Error:
  Game.create!(
    name: "Isogame",
    owner: current_player
  )

ActiveModel::UnknownAttributeError:
  unknown attribute 'owner' for Game.
```

When we created our migration file, we told the database that a game owner is a
player but we still haven't told Rails so, as far as Rails is concerned, there
is no such field as `owner`.

Go back to the model file (`app/models/game.rb`) and tell Rails that owner is an
association to the players table:
```ruby
class Game < ApplicationRecord
  belongs_to :owner, class_name: "Player"

  validates :name, presence: true
  validates :name, uniqueness: true
end
```

`belongs_to` means this record has only one associated record and the ID of that
association should be on this table. Specifically, a game has one owner and the
games table has a column called `owner_id`.

Test error:
```
Failure/Error: expect(new_game.players).to include player
     
NoMethodError:
  undefined method `players'
```

This means our previous test has passed but, before we move on, let's get the
game name from the form.

Rails provides the form values to controllers via the `params` method. Rather
than accessing the values directly, however, Rails uses a system called
[strong parameters](https://guides.rubyonrails.org/v6.0/action_controller_overview.html#strong-parameters)
to ensure that your users can only save a limited set of fields when setting a
request. For example, you wouldn't want it to be possible for someone to hijack
a form to assign themselves as the owner of someone else's game.

Change the controller to read:
```ruby
def create
  Game.create!(
    name: game_params[:name],
    owner: current_player
  )
end

private

def game_params
  params.require(:game).permit(:name)
end
```

Here, we are saying the only parameter that players can send is "name". When
we say `require(:game)`, it tells Rails to search for the parameters nested
under a "game" namespace. You will recall our inputs have names like
`game[name]`.

Putting the params logic into its own private method not only makes it reusable
but also helps to isolate the logic with an easy to understand method name.

Test error:
```
Failure/Error: expect(new_game.players).to include player
     
NoMethodError:
  undefined method `players'
```

The same error again—perfect, this means our change worked.

Now Rails is saying that games don't have a method called "players". This is
supposed to be the list of players in a game so let's create an association in
the game model:
```ruby
class Game < ApplicationRecord
  belongs_to :owner, class_name: "Player"

  has_and_belongs_to_many :players

  validates :name, presence: true
  validates :name, uniqueness: true
end
```

`has_and_belongs_to_many` tells Rails the relationship between games and players
is many-to-many. There can be many players in a game and players can play many
games.

Test error:
```
Failure/Error: expect(new_game.players).to include player

ActiveRecord::StatementInvalid:
  PG::UndefinedTable: ERROR:  relation "games_players" does not exist
  LINE 1: SELECT 1 AS one FROM "players" INNER JOIN "games_players" ON...
```

For the many-to-many relationship to work, there will need to be a database
table that records each time a player joins a game. The table will have a
`player_id` and a `game_id`. When you asked a game for its players, Rails will
look in the join table for all records for that game and it will search the
players table for the players in the game.

Creating the join table will require a new database migration:
```bash
rails g migration CreateGamesPlayers
```

Open the new migration (`db/migrate/[date]_create_games_players.rb`):
```ruby
# frozen_string_literal: true

class CreateGamesPlayers < ActiveRecord::Migration[6.0]
  def change
    create_table :games_players, id: false do |t|
      t.references :game, null: false, foreign_key: true
      t.references :player, null: false, foreign_key: true
      t.timestamps
    end
  end
end
```

We set `id: false` because a join table doesn't need a sequential primary key.
Those keys are used as a way to uniquely identify a record in the table but, in
the case of a join table, we can identify a record using the combination of the
game ID and player ID.

Run and then commit the migration with:
```bash
rails db:migrate
rubocop db
git add --intent-to-add db/migrate
git add --patch db
git commit
```

> Create games_players join table

Back to running tests:
```
Failure/Error: expect(new_game.players).to include player

  expected #<ActiveRecord::Associations::CollectionProxy []> to include #<Player id: 2, email: "player@example.com", created_at: "2020-09-27 04:36:45", updated_at: "2020-09-27 04:36:45">
  Diff:
  @@ -1 +1 @@
  -[#<Player id: 2, email: "player@example.com", created_at: "2020-09-27 04:36:45", updated_at: "2020-09-27 04:36:45">]
  +[]
```

Your error might look slightly different, but it should be inherently the same.
It is saying that, when it asked a game for its players, it expected to receive
a collection that includes only the game owner but, instead, the collection was
empty.

Let's add the player to the game when the game is created in the controller:
```ruby
def create
  game = Game.create!(
    name: game_params[:name],
    owner: current_player
  )
  game.players << current_player
end
```

## Create a template for the game page

Test error:
```
Failure/Error: show_game_page = Pages::Games::Show.new
     
NameError:
  uninitialized constant Pages::Games::Show
```

The game is now being created in the database and Rails is trying to show the
page to the player. We need a new page class for testing showing the game.

Create `spec/pages/games/show.rb`:
```ruby
# frozen_string_literal: true

module Pages
  module Games
    class Show < SitePrism::Page
    end
  end
end
```

Test error:
```
Failure/Error: expect(show_game_page).to be_displayed(id: new_game.id)
     
SitePrism::NoUrlMatcherForPageError:
  SitePrism::NoUrlMatcherForPageError
```

We need to tell SitePrism what the URL will be. The way Rails knows which game
to show is by looking for the ID of the game in the URL in the format
`/games/[id]`. Similar to how we passed token into the Devise `ResetPassword`
and `Confirmation` page classes, we pass that ID as a parameter in the URL:
```ruby
class Show < SitePrism::Page
  set_url "/games/{id}"
end
```

Test error:
```
Failure/Error: expect(show_game_page).to be_displayed(id: new_game.id)
```

We are expecting to see a page showing the newly created game but that is not
what is happening. This is because we haven't told Rails to perform that
redirect and we are still on the `create` page.

In the controller:
```ruby
def create
  game = Game.create!(
    name: game_params[:name],
    owner: current_player
  )
  game.players << current_player

  redirect_to game
end
```

Rails can see that `game` is a game that has been stored in the database and so
interprets `redirect_to game` as "show the new game page".

Test error:
```
Failure/Error: redirect_to game

NoMethodError:
  undefined method `game_url' for #<GamesController:0x00007fcb89c0c900>
  Did you mean?  games_url
```

This is another of those obscure routing errors. Rails has been told that games
have `new` and `create` actions but we didn't provide a `show` action. Open
`config/routes.rb` and add the new action:
```ruby
resources :games, only: %i[new create show]
```

Test error:
```
Failure/Error: new_game_page.create_button.click
     
AbstractController::ActionNotFound:
  The action 'show' could not be found for GamesController
```

Back in the controller, add a `show` method:
```ruby
def new
  @game = Game.new
end

def create
  ...
end

def show
end
```

Test error:
```
Failure/Error: new_game_page.create_button.click
     
ActionController::MissingExactTemplate:
  GamesController#show is missing a template for request formats: text/html
```

Create a template for the new action at `app/views/games/show.html.erb`:
```erb
<%= @game.name %>
```

That basic page will do nothing more than show the name of the game; enough to
prove it's working.

Test error:
```
Failure/Error: <%= @game.name %>
     
ActionView::Template::Error:
  undefined method `name' for nil:NilClass
```

There is that `nil` error again. In this case, we are sending the `name` message
to `@game` but `@game` is `nil`. We need to set `@game` in our controller:
```ruby
def show
  @game = Game.new
end
```

Test error:
```
Failure/Error: expect(show_game_page).to have_content("Isogame")
  expected to find text "Isogame" in "Cards Against Isolation"
```

"Isogame" was the name we set for the game. The page is now displaying
`@game.name` but the name being shown is not "Isogame". This is obviously
because `@game` is just a new game, not the game we created.

Rails gave us access to form values in `create` requests using the `params`
method. For `show` requests, it uses `params` to expose permitted values in the
URL. We can find the newly created game using:
```
def show
  @game = Game.find(params[:id])
end
```

## Avoiding duplicates

That should complete the "it creates the game and redirects to the game page"
test. Onto "when the new game name matches an existing game".

Test error:
```
Failure/Error:
  game = Game.create!(
    name: game_params[:name],
    owner: current_player
  )

ActiveRecord::RecordInvalid:
  Validation failed: Name has already been taken
```

It's good to see that Active Record is not allowing duplicate records but,
rather than showing an error message, the page is blowing up. The reason why
an exception is thrown is because we're using the bang in the method name,
`create!`. Both `create!` and `save!` will throw an exception if there is an
error while `create` and `save` will not. I always default to using the bang
methods so there is no chance for silent errors but, in this case, it would make
more sense to use the non-bang method and check the response (`true` means the
record was saved while `false` means there was an error).

Update the method to:
```ruby
def create
  game = Game.new(
    name: game_params[:name],
    owner: current_player
  )

  if game.save
    game.players << current_player
    redirect_to game
  else
    render "new"
  end
end
```

Now, if there is an error, Rails will go back to the new game page rather than
the show game page.

Test error:
```
Failure/Error: expect(new_game_page).to have_error_message
  expected #<Pages::Games::New:0x00007f92f1dd4b28> to respond to `has_error_message?`
```

`error_message` was just a label we came up with to some message on the page but
we haven't actually defined what an error message is.

This element can be defined in `spec/pages/games/new.rb`:
```ruby
class New < SitePrism::Page
  set_url "/games/new"

  element :error_message, ".alert"

  element :name_field, "input[name='game[name]']"
  element :create_button, "input[type=submit]"
end
```

Test error:
```
Failure/Error: expect(new_game_page).to have_error_message
  expected #<Pages::Games::New:0x00007f8738161d38 @loaded=true, @load_error=nil> to have error message
```

Now that our test knows where to go looking for errors, it is telling us there
are no errors. We will be showing the errors in the flash messages. When the
errors are encountered in the `create` action, we need to pass the error
messages as flash messages. Formatting error messages is really outside the
main task of creating a game so we'll do this in a separate method:
```ruby
def create
  game = Game.new(
    name: game_params[:name],
    owner: current_player
  )

  if game.save
    game.players << current_player
    redirect_to game
  else
    render_form_errors(game)
  end
end

def show
  @game = Game.find(params[:id])
end

private

def game_params
  params.require(:game).permit(:name)
end

def render_form_errors(game)
  error_messages = game.errors.full_messages.join(". ")
  flash[:alert] = error_messages

  render "new"
end
```

`game.errors.full_messages` returns an array of errors. Since there is only one
error on this form, the array looks like:
```ruby
["Name has already been taken"]
```

All we are doing is placing all the errors on one line separated by a full stop
and space. If there were 3 errors like:
```ruby
["First error", "Second error", "Third error"]
```
The result would be:
```
First error. Second error. Third error
```

## Finishing up

Now we need to commit all that work but it seems Rubocop is upset about one of
the tests having too many lines. I generally agree that tests should be short
but I have a bit more flexibility for feature specs because they can be _really_
slow and so it can be valuable to test a few things in one test.

Open `.rubocop.yml` and then find and adjust `RSpec/ExampleLength`:
```yaml
RSpec/ExampleLength:
  Exclude:
    - spec/features/**/*
  Max: 10
```

Now commit that change:
```bash
git add --patch .rubocop.yml
git commit
```

> Remove the Rubocop maximum lines rule for feature specs
>  
> Because feature specs can be slow, it can be helpful to test many things  
> in each test. This increases the number of lines in the tests

Now the remaining files can be committed:
```bash
rubocop
rspec
git add --intent-to-add .
git add --patch .
git commit
```

> Allow players to create games

If you had an issues, you can
[review the code](https://github.com/HashNotAdam/cards-against-isolation/commit/e3172d1b550fb3e4b1d4d3d62045514218bf4772).
