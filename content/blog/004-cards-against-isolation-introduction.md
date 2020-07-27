---
title: "Cards Against Isolation (Introduction)"
date: 2020-07-28T09:18:47+10:00
slug: "cards-against-isolation-introduction"
description: ""
keywords: []
draft: false
tags: [development, tutorial]
math: false
toc: false
---
I have a lot of love for
[Cards Against Humanity](https://cardsagainsthumanity.com/)—it's a hell of a
lot of fun on a Saturday night after a few drinks with friends. Unfortunately,
the online version is a disgrace; the interface was rushed, it doesn't include a
lot of cards so you need to expect duplicates, and it is lacking in some really
basic features.

Since it's rather clear that the current pandemic isn't leaving us any time
soon—I write during Melbourne's second lockdown—I thought it might be fun to
create my own version of the game.

I'll post the progress here as a tutorial with a focus on how and why I make
decisions. There is nothing I appreciate more from developers than strong
reasoning for their choices. The problem, however, is that most tutorials
promote copy-and-paste without context rather than helping to provide the
foundational knowledge needed to make your own choices. I look forward to people
following along and telling me what they would do differently.

## Planning

Building an application with the complexity of a real-time card game can seem
daunting at first, but possibly the greatest attribute of good developers is the
understanding that any big problem is really just a series of small problems.
The best way to make this leap is with planning. As my fitness coach always
tells me, "proper planning and preparation prevents piss poor performance".

Cards Against Humanity is played with at least 3 players. Technically there is
no upper limit, however, I can’t even imagine how slow and frustrating it would
be with large numbers. My suggestion is to keep it to a maximum of 2 pizzas (8
people). This is an arbitrary number which you can increase should you enjoy
life’s great monotonies.

Each player has 10 white (answer) cards. One player will be the judge and they
read out a black (question) card which includes one or more “blanks”. Once all
answers have been submitted, the judge picks a winner who receives a point.

There are other rules and possible variations, but the plan is to first produce
a simple version of the game and then, once it’s deployed and running,
we can always build upon that foundation.

I’m going to stick with Ruby on Rails for this project. I find the Ruby language
to be expressive and a real joy to write. The Rails framework is not only easy
to learn and very well supported but it allows you to focus your effort on your
application logic rather than the technology. This always makes me happy when
I’m building business products (whether client work or internal apps). I’d much
rather spend my time solving business problems than programming problems.

Given we need to manage state (there could be many games with many players with
many cards with many points) we are going to need some kind of data store—the
simplest answer to this question is a relational database. I’ve chosen to use
PostgreSQL—it’s an extremely well proven database that’s relatively easy to get
up and running but, if you have any issues, the community is huge and so it’s
easy to get support. If you are already familiar with MySQL, feel free to use
that instead. Ruby on Rails, via Active Record, creates a consistent interface
for managing data so, while I have strong opinions about which database to use
in a serious business app, you would be unable to tell the difference between
databases in a little project like this.

That’s the back-end at a high level, now what about the front-end?

One of the challenges with a multi-player card game is that there needs to be a
way to know when another player makes a move. In the context of Cards Against
Humanity, after select your answer/s, you then wait for the judge to decide
upon a winner. Once they have chosen, a new question card will be presented. A
standard HTML website would have no way of knowing that there is new information
to receive and so it would be up to the user to refresh the page until something
changes. This wouldn’t make for a particularly great user experience.

One way to introduce real-time, bi-directional messaging is with WebSockets.
There is a lot involved in setting up and managing a socket but Rails has a
WebSockets library called Action Cable that does the bulk of the heavy lifting.
Even with Action Cable, however, having elements react to real-time data is
involved and requires a lot of JavaScript (ew!). Thankfully, there are some
clever new packages that integrate with Action Cable to make this a reasonably
painless process. I wouldn’t normally add a new and unproven package into
production code but fun projects like this are the perfect place to get outside
our comfort zone. I’ve chosen to go with
[Motion](https://github.com/unabridged/motion). I also considered
[StimulusReflex](https://github.com/hopsoft/stimulus_reflex) which is getting a
lot of buzz at the moment but I felt that the way it "magically" injected
instance variables into your controller was a bit jarring and I really wanted to
use [ViewComponent](https://github.com/github/view_component), which Motion
supports. ViewComponent allows you to create reusable and testable components.
It's really common for views to be hard to follow and include variables and
helper methods that are difficult to debug. Joel Hawksley gave fantastic talk
about this at RailsConf 2019 called
[Rethinking the View Layer with Components](https://www.youtube.com/watch?v=y5Z5a6QdA-M).

### User journey and data modelling

At this point in the process, I wouldn't expect to have a complete and final
view of how the application is going to work and how data will be stored but I
certainly want to be considering it because it will fundamentally change the way
I build.

At a high level we know that we need:
- Games (there could be many games happening at the same time as well as many
games played over time)
- Players
- Rounds (the current state of a game)
- Question cards
- Answer cards
- Points

When it comes to players, it's important to consider whether they log in or if
people can play without an account. The official Cards Against Humanity game
does not require accounts. The advantage of this is simplicity and ease of mind.
Noone ever worries about how their data is being used and they can jump into a
game with a simple click of a link. On the other hand, it means you have no way
of knowing that someone on a phone and a computer is the same person. I've had
situations where someone has started on their computer but then decided it would
be better to use their phone (this is also because the official game is
basically unusable on computer) and you end up with duplicate players. You could
absolutely argue either way on this but I'm going to implement accounts since it
also means we could also have a leader board. That decision has big
ramifications for how the interface will work so considering it early is
important.

To keep the complexity down, when players first come to the site we’ll show them
a very basic page with a login form and a link to a page to register. The
[Devise gem](https://github.com/heartcombo/devise) can do most of the work here.
Devise handles registration, login, forgotten passwords, email notifications,
etc.

Once players have logged in, we will show them:
- games they created or have joined;
- games that have not yet started that they can join; and
- the option to create a new game.

There is very little we need to know about a player. Purely from the perspective
of a game, we really just need to call them something. Of course, Devise is
going to need a few other things (email, password, etc) but the great thing
about Devise is that it will take care of all that.

We can also keep games really simple. All we need to know is who created it,
which players are in the game, and the state of the game (pending, active,
finished).

Since players have accounts, they could end up playing many games. This means
that players have many games and games have many players—this is known as a
many-to-many relationship. To map a many-to-many relationship in a database, you
need to have a join table that keeps a record of every game that a player is
involved in—we can call this Player Games. Thankfully, Rails and Active Record
manage this for us too.

When a game is started, we'll need to allocate question and answer cards. This,
of course, means that we need to store all of the potential questions and
answers somewhere. Your first instinct may be to have a cards table with a type
field (either question or answer). I think this would be a mistake that would
come back to bite you because, while there is a lot of similarity between these
concepts, they are not the same thing and they might diverge as you learn more.
One example is that question cards will need some concept of “blanks” for
answers to fill. Most likely, this will require some logic that isn’t necessary
for answer cards. Rails Single Table Inheritance (STI) tries to solve this
problem, allowing you to have separate classes (and therefore separate logic)
for each type. The problem is that, if we later decide that question cards need
an extra column for functionality we haven’t yet considered, we are forced to
have that same field available to answer cards. At first, however, we want to
keep it simple: we’ll have a questions table with a text column and an answers
table with a text column.

Now that there is a concept of question and answer cards, we will still need to
allocate answer cards to the players. We can have a Player Answer Cards table
which has one record per player per card. These records will also need to have a
connection to the game, otherwise a player who is in multiple games
simultaneously would only have 1 hand of cards shared between the games. How do
we know if the card has been played, though? We’ll need a reference to the round
it’s used in. You might recall that some questions require 2 answer cards so I
guess we’ll need a position so we know the order they should be in.

If a player answer card is associated with a round, that means we need a round
table. Rounds are associated with games; they have judges (the player who is
judging the round); and there is a winning player.

### Build approach

Now that we have an idea of how this is going to be structured, there are many
ways we could choose to build it. One way might be to start back-end heavy,
building out the data models and the game logic and then plugging in an
interface later. While I’m a full stack developer, the majority of my work
involves back-end logic and so I’m naturally more comfortable there and the
back-to-front approach is appealing. The problem with spending so much time in
the back-end first, however, is that it can be a long time before you get any
real feedback (not that the red/green colours of your test suite aren’t super
exciting). Also, if you build by feature, your git commits are going to tell a
more cohesive story.

At a high level, the features are going to be:
- authentication;
- create a game;
- join a game;
- start a game;
  - allocate answer cards; and
  - create a round
    - assign a judge; and
    - assign a question card
- show players the question card;
- allow players to select an answer;
- ask the judge to pick the best answer;
  - store the winner on the current round;
  - announce the winner;
  - increment the points; and
  - create a new round
- when all the questions have been asked, finish the game

Let's getting a new project started **(coming really soon)**
