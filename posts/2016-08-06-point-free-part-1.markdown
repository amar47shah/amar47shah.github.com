---
title: 'Point-Free or Die: Part 1'
---

_This is the first post in a series about **tacit programming** and **point-free**
syntax in Haskell and other languages._

For an initial understanding of [tacit programming](https://en.wikipedia.org/wiki/Tacit_programming),
consider that a synonym for tacit is **quiet** and an antonym is **noisy**.
Code is noisy when it contains more information than we need to see.
It's quiet...when it doesn't.

One way of producing tacit code is to write function definitions in a **point-free**
style.

As opposed to a **point-ful** definition, a point-free definition is tacit
because it tells you about the function without mentioning one or more of its arguments.

Here is an example of a point-ful definition:

    lengths ls = map length ls

Here is its point-free equivalent:

    lengths = map lengths

These definitions are semantically equivalent; the only difference is the syntax.

Both of them have this type:

    lengths :: Foldable t => [t a] -> [Int]

In other words, `lengths` receives a list of foldable structures containing values
of some generic type and returns a list of `Int`.

The point-ful and point-free definitions mean the same thing, but they
communicate differently.

You could read the first definition as:

"`lengths` is a function that maps each foldable structure in a list to its `length`."

You could read the second as:

"`lengths` is a map of `length`."

Consider the succinctness of the second version. It's shorter and quicker to say,
but understanding it requires that you how `map` acts on the elements of a list.

## Calibrating Abstraction

Here's another example:

    sum xs = foldr (+) 0 xs

"`sum` is a function that iterates over `xs` to add up its elements, starting
from zero and proceeding from right to left."

Or you could say:

    sum = foldr (+) 0

"`sum` is a right-fold of addition starting from zero."

Both define the same function. But when we leave out the `xs`, we communicate
at a higher level of abstraction.

**We are no longer talking about what `sum` does, but what it is:**
a fold with certain properties.

Let's combine these two functions to make a new one:

    totalNumber ls = foldr (+) 0 (map length ls)

By substituting in our definitions for `sum` and `lengths`, we get

    totalNumber ls = sum (lengths ls)

And of course, we can make this look a little more like Haskell by using `$`.

    totalNumber ls = sum $ lengths ls

You could say it's quieter with the `$`, but it's not point-free yet. `totalNumber`
is defined in relation to its argument `ls`: "`totalNumber` receives a list of
foldable structures and returns the `sum` of the `lengths` of those structures."

But then, you could also just say, "`totalNumber` is the **composition** of `sum`
and `lengths`." How do we get there?

    totalNumber ls =        sum $ lengths ls
    totalNumber ls =        sum . lengths $ ls
    totalNumber    = \ls -> sum . lengths $ ls
    totalNumber    =        sum . lengths

This is the process of **eta-reduction**: converting a point-ful definition to a
point-free one. Here it takes three steps:

1. First, we notice that applying `sum` to `lengths ls` is equivalent to applying
the composition `sum . lengths` directly to `ls`.
2. Next, we consider that `totalNumber` is a function that receives `ls` and
simply applies an expression to it.
3. Finally, we drop the unnecessary lambda, removing `ls` from the definition.

Eta-reduction, also called [eta-conversion](https://en.wikipedia.org/wiki/Lambda_calculus#.CE.B7-conversion)
is a simply mechanical process--which means computers can do it!

In fact, you don't have to do any of your eta-reduction yourself.
Haskell's [`pointfree`](https://github.com/bmillwood/pointfree) library can do it for you!


