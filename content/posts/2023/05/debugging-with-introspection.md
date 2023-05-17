---
title: "Debugging with introspection"
date: 2023-05-17T09:00:00+10:00
slug: "debugging-with-introspection"
description: "Unleashing the Power of method and $"
keywords: []
draft: false
tags: [development]
math: false
toc: false
---

Debugging is an essential skill for developers, and Ruby provides powerful tools to help identify
and fix issues in code. While most Ruby developers are familiar with tools like debug, byebug,
pry-byebug, and the classic puts/p/pp, I would like to explore alternatives powered by
introspection.

Ruby is a dynamically typed and reflective programming language, which means it provides built-in
features that enable introspection and self-modification of code. These features contribute to
Ruby's ability to inspect and manipulate its own structures and behaviour at runtime. Here are some
key aspects of the Ruby language that make introspection possible:

### Reflection
Reflection is a fundamental concept in Ruby that allows a program to examine and modify its own
structure and behaviour. Ruby provides a rich set of reflection capabilities, including the ability
to access class and object information, retrieve method details, and dynamically modify classes and
objects.

```ruby
class Neighbour
  attr_accessor :name, :age

  def greet
    puts "Hello, #{name}!"
  end
end

# Creating an instance of MyClass
obj = Neighbour.new
obj.name = "Maddison"
obj.age = 25

# Reflection example
puts "Class name: #{obj.class}"                      # Output: Class name: Neighbour
puts "Instance variables: #{obj.instance_variables}" # Output: Instance variables: [:@name, :@age]
puts "Methods: #{obj.methods - Object.methods}"      # Output: Methods: [:age, :greet, :age=, :name=]

# Modifying object behavior dynamically
obj.public_send(:greet)  # Output: Hello, Maddison!
```

In this example, we define a class, `Neighbour`, with two instance variables (name and age) and a
greet method. Through reflection, we can obtain information about the class name (`obj.class`),
instance variables (`obj.instance_variables`), and methods (`obj.methods`). We use the `public_send`
method to dynamically invoke the greet method on the `obj` object.

Reflection is a powerful feature that allows Ruby programmers to write flexible and dynamic code,
making it possible to perform various operations at runtime based on the structure and behavior of
objects.

### Metaprogramming
Metaprogramming refers to the ability of a programming language to write code that generates or
modifies code. In Ruby, metaprogramming is supported through features such as dynamic method
creation, runtime class modification, and the ability to define methods and classes dynamically.
These capabilities enable developers to introspect and modify code structures during program
execution.

```ruby
class FooBarBaz
  # Define dynamic methods using metaprogramming
  %i[foo bar baz].each do |method_name|
    define_method(method_name) do
      puts "Called #{method_name}"
    end
  end
end

# Create an instance of FooBarBaz
obj = FooBarBaz.new

# Call dynamic methods
obj.foo  # Output: Called foo
obj.bar  # Output: Called bar
obj.baz  # Output: Called baz
```

Here, we define the class `FooBarBaz`. Using metaprogramming, we dynamically generate methods based
on an array of symbols. The `define_method` method is used to define each method dynamically. In
this case, the methods `foo`, `bar`, and `baz` are created, and when called, they simply print a
message indicating which method was invoked.

### Dynamic Typing
Ruby is a dynamically typed language, which means variables do not have a fixed type and can change
at runtime. This flexibility allows developers to introspect and manipulate objects and their types
dynamically. For example, you can check an object's class, retrieve and modify its instance
variables, and dynamically invoke methods based on runtime conditions.

### Method Objects
Ruby treats methods as first-class objects. This means that methods can be assigned to variables,
passed as arguments to other methods, and returned as values. Method objects enable powerful
introspection and manipulation capabilities, allowing you to examine method details, call methods
dynamically, and redefine method behaviour.

```ruby
class Greeter
  def greet(name)
    puts "Hello, #{name}!"
  end

  def call_greet(callback)
    callback.call("Alice")
  end
end

# Create an instance of Greeter
obj = Greeter.new

# Create a method object for the "greet" method
greet_method = obj.method(:greet)

# Invoke the method object
greet_method.call("John") # Output: Hello, John!

# Pass the method around as a call back like this is JavaScript
obj.call_greet(greet_method) # Output: Hello, Alice!
```

In this example, we define a class, `Greeter`, with a `greet` method that takes a name parameter and
prints a greeting message. We create an instance of `Greeter` and obtain a method object for the
greet method using the `method` method, which takes the method name as a symbol. The resulting
method object, `greet_method`, can be assigned to a variable or passed around like any other object.
We can then invoke the method object using the `call` method and provide the necessary arguments.

### Reflection APIs
Ruby provides a comprehensive set of reflection APIs that expose the language's internal structures
and metadata. These APIs, such as the `Object#class`, `Module#instance_methods`, and `Method` class,
allow developers to access information about classes, objects, modules, methods, and more. With
these APIs, you can retrieve information about method names, parameters, visibility, source
locations, and perform various introspection tasks.

### Open Classes
Ruby's "open classes" feature allows you to modify existing classes and add or redefine methods
dynamically. This capability plays a significant role in introspection and metaprogramming, as it
enables developers to extend and modify existing classes to suit their needs. By reopening classes
and modifying their behaviour, you can introspect and manipulate the runtime behaviour of objects
and classes.

```ruby
# File 1
class Greeter
  def greet
    puts "Hello, #{name}!"
  end
end

# File 2
class Greeter
  def greet
    puts "Howdy, #{name}!"
  end
end

Greeter.new.greet("Adam") # Output: Howdy, Adam!
```

## Setting the scene

These features collectively make Ruby a highly introspective language, empowering developers to
examine and modify code structures, inspect objects and classes, dynamically invoke methods, and
redefine behaviours at runtime. This introspective nature provides a powerful foundation for
metaprogramming, debugging, testing, and building dynamic and flexible applications.

Understanding how to leverage these features will greatly enhance your ability to diagnose and
troubleshoot Ruby code effectively.

Let's say you are debugging some code that contains some indirection. Maybe the method being called
is in a gem or maybe the receiver has been monkeypatched and you're not sure which code will
receive the message.

One option could be to use pry-byebug to step through the code, but I've found there are many
situations where this becomes flaky and difficult to use. You might also already be in a console
and, while adding a breakpoint isn't particularly time consuming, it feels heavy handed. Having
alternatives in your toolkit can really level up your debugging powers.

Let's say you have a Cat class that speaks:
```ruby
class Cat
  def speak
    "meow"
  end
end
```

When you call this method, however, you get an unusual response:
```ruby
> Cat.new.speak
=> "woof"
```

How might you get to the bottom of this? (Other than scratching your head and hoping for the best)

## The $ Global Variable in IRB / Rails console

In interactive Ruby (IRB)—and therefore also in the Rails console—`$` can show you the definition of
a method and its location.

In the case above, we could find the message receiver by running:
```ruby
> $ Cat.new.speak
```

The response we will receive is the path and line number of the file that contains the method and
the definition itself:
```ruby
From: [project path]/config/initializers/monkey_the_cat.rb:2

  def speak
    "woof"
  end
```

## The method Method

Most of the time, `$` is going to be your best option, however, for more advanced requirements, the
`method` method can step it up a notch.

On the surface, it appears to offer similar functionality but with a bit of indirection:
```ruby
> Cat.new.method(:speak)
=> #<Method: Cat#speak() [project path]/config/initializers/monkey_the_cat.rb:2>
```

While you can't immediately see the method definition, any decent console will allow you to click
the path to open the file in your editor (command-click on macOS).

So, what can the `method` method do that `$` can't?

### Inspecting Method Details
The `method` method is particularly useful when you want to obtain detailed information about a
specific method. By retrieving a `Method` object using `method(:method_name)`, you can access
information such as the method's name, owner, receiver, parameters, and source location. This level
of introspection can be valuable for understanding the behaviour and characteristics of a method,
especially when you need to troubleshoot or analyse its implementation.

### Dynamic Method Invocation
If you need to invoke a method dynamically based on runtime conditions, the `method` method provides
a convenient way to do so. By calling `method(:method_name).call(arguments)`, you can execute a
specific method based on its name stored in a variable or determined at runtime. This dynamic method
invocation can be helpful when you want to conditionally execute different methods based on certain
criteria during debugging or experimentation.

### Method Redefinition
When you need to modify or redefine a method at runtime for debugging or testing purposes, the
`method` method allows you to obtain a `Method` object representing the original method. You can
then redefine the method using techniques like method aliasing or method overriding. This approach can be
handy when you want to temporarily modify a method's behaviour for testing different scenarios or
diagnosing specific issues without permanently altering the original code.

### Advanced Method Manipulation
The `method` method opens up possibilities for more advanced method manipulation techniques. For
example, you can bind a method to a different object using the `unbind` and `bind` methods of the
Method object. This allows you to change the method's receiver dynamically, which can be helpful for
testing or debugging scenarios where you want to simulate different contexts or modify the behaviour
of a method at runtime.

## Conclusion
Debugging is a crucial skill for developers, and Ruby provides powerful features like the `$` global
variable in IRB and the `method(:method_name)` method to assist in the debugging process. By
utilising these features effectively, you can gain better insights into the flow of execution,
inspect objects and variables, track control flow, and introspect method behaviour. Armed with these
techniques, you'll be better equipped to debug Ruby code and resolve issues efficiently.

Happy debugging!
