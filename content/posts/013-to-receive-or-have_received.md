---
title: "To receive or have_received"
date: 2023-01-05T12:00:00+11:00
slug: "to-receive-or-have_received"
description: "Why I'm preferencing an upfront expectation in RSpec over using have_received"
keywords: []
draft: false
tags: [development]
math: false
toc: false
---
In [99 Bottles of OOP](https://sandimetz.com/99bottles), Sandi Metz et al.
describe the three parts to a test as:

- Setup
- Do
- Verify

Consistently using this pattern makes tests easier to read and write and, while
the idea existed before 99 Bottles, Metz is very much the guiding light for the
Ruby community.

Saying this, I've decided that, due to the way RSpec works, I believe testing
that messages were received should be an exception to this rule.

If you are using [RuboCop RSpec](https://github.com/rubocop/rubocop-rspec), you
will be told to write your tests like this:
```ruby
allow(SomeObject).to receive(:a_message).with(:arguments)

run_the_code

expect(SomeObject).to have_received(:a_message)
```

This follows the steps discussed in 99 Bottles and contributors to RuboCop RSpec
reference Metz's talk,
[The Magic Tricks of Testing](https://www.youtube.com/watch?v=URSWYvyc42M),
when justifying the Rubocop rule.

```ruby
# Setup: Create the environment required for the test by spying on SomeObject
allow(SomeObject).to receive(:a_message).with(:arguments)

# Do: Perform the action to be tested
run_the_code

# Verify: Confirm that SomeObject received the message
expect(SomeObject).to have_received(:a_message)
```

The problem comes when multiple of the same message are sent to an object with
different arguments:
```ruby
allow(SomeObject).to receive(:a_message).with(:argument_1)
allow(SomeObject).to receive(:a_message).with(:argument_2)
allow(SomeObject).to receive(:a_message).with(:argument_3)

run_the_code

expect(SomeObject).to have_received(:a_message).exactly(3).times
```

The trap that is easy to fall into is that this is only testing that the message
is sent 3 times and that, in each case, either `argument_1`, `argument_2`, or
`argument_3` are sent. It is not validating that all three arguments were
passed. The test will pass even if our code reads:
```ruby
SomeObject.a_message(:argument_1)
SomeObject.a_message(:argument_1)
SomeObject.a_message(:argument_1)
```

This can be solved by listing the arguments in the expectation:
```ruby
allow(SomeObject).to receive(:a_message).with(:argument_1)
allow(SomeObject).to receive(:a_message).with(:argument_2)
allow(SomeObject).to receive(:a_message).with(:argument_3)

run_the_code

expect(SomeObject).to have_received(:a_message).
  with(:argument_1).
  with(:argument_2).
  with(:argument_3)
```

While this is test is functionally correct, it is unnecessarily verbose. The
bigger problem I see, however, is that there isn't a simple way for engineers to
realise the previous test has a problem. When you read the documentation or
[examples](https://github.com/rubocop/rubocop-rspec/issues/268#issuecomment-273014781)
given when defending the rule, this issue isn't discussed. I can't even begin to
tell you how many hours I spent researching why my tests were incorrectly
passing.

While it is technically incorrect—and a little jarring initially—to begin a
spec with an expectation, test accuracy is surely more important than adhering
to the rules. I've updated by Rubocop config to include:
```yaml
RSpec/MessageSpies:
  EnforcedStyle: receive
```
and would now write the above test like:
```ruby
expect(SomeObject).to receive(:a_message).with(:argument_1)
expect(SomeObject).to receive(:a_message).with(:argument_2)
expect(SomeObject).to receive(:a_message).with(:argument_3)

run_the_code
```

I've also found, after making that change, that Rubocop RSpec will try to get
you to create an `allow` statement when using a return value like:
```ruby
allow(SomeObject).to receive(:a_message).with(:argument_1).and_return(:foo)
expect(SomeObject).to receive(:a_message).with(:argument_1)
```
So my config now also includes:
```yaml
RSpec/StubbedMock:
  Enabled: false
```
