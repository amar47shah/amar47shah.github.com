---
title: 'Point-Free or Die, Part 2: Wholemeal Programming'
---

_This is the second post in a series about **tacit programming** and **point-free**
syntax in Haskell and other languages.
Be sure to check out [Part 1](/posts/2016-08-06-point-free-part-1.html)!_

We left off in [Part 1](/posts/2016-08-06-point-free-part-1.html)
having built three functions: `lengths`, `sum`, and `totalNumber`.
Here are their point-free definitions:

    lengths :: Foldable t => [t a] -> [Int]
    lengths = map length

    sum :: (Num a, Foldable t) => t a -> a
    sum = foldr (+) 0

    totalNumber :: Foldable t => [t a] -> Int
    totalNumber = sum . lengths

Notes on types:

1. `lengths` returns a list of `Int`.
2. `Int` is a numeric type, because it implements the `Num` typeclass.
3. Lists are `Foldable` containers.
4. `lengths` returns `[Int]`, a foldable container of values of a numeric type.
5. `sum` receives a foldable container of values of a numeric type.
6. Therefore, `sum` can receive the output of `lengths`.

Moreover,

1. `totalNumber` receives the same type of input as `lengths`.
2. `totalNumber` returns an `Int`, which is a type that `sum` can return.

All of this is exactly as it should be, since `totalNumber` is a **composition**
of `sum` and `lengths`. Composition, that little dot, is a function, too:

    (.) :: (b -> c) -> (a -> b) -> a -> c
    (.) = \f g x -> f (g x)

The `.` operator is a **combinator**, a lambda expression that references
no variables other than its own arguments. You can imagine that `.` takes
two functions and lines them up in a pipe that goes from _right to left_.

`totalNumber` is a pipe that **transforms** a list of foldable containers
into a single integer. When we describe it as a composition, we don't have
to explicitly walk through each "phase" of the transformation.
We can be **tacit** about all that.

---

In my mind, there are a couple of problems with `totalNumber`:

First, `sum . lengths` is self-explanatory: it's the _sum of lengths_.
Renaming it to `totalNumber` doesn't add much value. I could just opt to use
the whole expression whenever I need a sum of lengths. And that could make for clearer code,
since there would be no need to look up the definition of `totalNumber`.

Second, `totalNumber` is oddly specific. If our data is a list of containers,
the total number of items is _just one of many_ ways to summarize it.

It would be great if we knew something more general about the many ways
to summarize a list of containers of items. Then we could derive more specific
transformations, like `totalNumber`, as we needed them.

For instance, we could have functions like these

* `totalMatch`: the sum of the numbers of _items that fulfill some requirement_ in each container.
* `totalMode`: the sum of the numbers of _occurrences of the most common item_ in each container.
* `totalMax`: if the contained items are themselves numbers, the sum of the _maximum values_ in each container.

## Wholemeal Programming

Something I love about functional programming is that it encourages you to think
about how problems can be generalized. This directly contradicts my experiences
with object-oriented programming, where YAGNI--You Ain't Gonna Need It--is a watchword.

Many times, I've ripped out Ruby code that was prematurely generalized to
handle never-used cases that complicated the implementation.
With pure functions, immutable data, and [Hindley-Milner parametric polymorphism](
http://akgupta.ca/blog/2013/05/14/so-you-still-dont-understand-hindley-milner/),
Haskell can provide general solutions that are often _no more complicated_ than
a specialized solution. Instead of tailoring your code to fit all the circumstances
you can imagine, you broaden it so that it can always operate wherever the _essential_
circumstance appears.

This is called [wholemeal programming](http://www.cs.ox.ac.uk/ralf.hinze/publications/ICFP09.pdf). Ralf Hinze writes:

> Wholemeal programming means to think big: work with an entire list,
> rather than a sequence of elements; develop a solution space,
> rather than an individual solution; imagine a graph,
> rather than a single path. The wholemeal approach often offers
> new insights or provides new perspectives on a given problem.
> It is nicely complemented by the idea of projective programming:
> first solve a more general problem, then extract the interesting
> bits and pieces by transforming the general program into more
> specialised ones.

Let's return to our overly specialized function, `totalNumber`. What's the
general problem we're trying to solve? Considering the potential uses
we brainstormed for a generalized "adding-up" function,
perhaps we could put it this way:

> We want to add up _some metric_ derived from each container's items.

In the case of `totalNumber`, that metric is the `length`. Therefore,
we can replace our old definition,

    totalNumber = sum . lengths

which we could have written,

    totalNumber = sum . map length

with a new one:

    totalNumber = aggregate length
            where aggregate f = sum . map f

`aggregate` abstracts the idea of _measuring each container and adding up the results_.
To use it elsewhere, let's promote its scope. It's good practice to write out
its type as well.

    aggregate :: Num c => (a -> c) -> [a] -> c
    aggregate f = sum . map f

And we might say:

> `aggregate` is a composition of `sum` with a mapping of some function `f`.

We're calling the argument `f` to indicate that it's a function. As you can see,
`f` has the type `a -> c`. It receives a value of type `a` and returns a value
of `c`, a numeric type.

`aggregate` receives a function of type `a -> c` _and then_ a list of `a`,
returning a single numeric `c`.

Whereas `totalNumber` only understands `length`, `aggregate` is generalized
to work with any metric!

## Projective Programming

We have a general solution. Let's project it onto a specialized problem:

> the sum of the maximum values in each container

Haskell's `Prelude` gives us `maximum`:

    Prelude> :t maximum
    maximum :: (Ord a, Foldable t) => t a -> a
    Prelude> maximum [4,6,9,2,-3,2,0]
    9

so how should we get the sum of maximum values of sublists in this list:

    totalMax [[6,-1,3], [-2,-1,-4], [1,1,1], [9,10,11]]

You guessed it. We'll use `aggregate`:

    totalMax :: (Num c, Ord c, Foldable t) => [t c] -> c
    totalMax = aggregate maximum

And yes, that's a point-free definition. What does `totalMax` do?

> `totalMax` aggregates `maximum`.

But we can also write `totalMax` in a point-ful way:

    totalMax cs = aggregate maximum cs

That is, we can write it in a way that communicates at a lower level:

> `totalMax` takes a list of containers and outputs the sum of their max values

## One Last Look

Earlier, I said that `aggregate` receives _two inputs_: a function that measures
a container, and also a list of containers. But in our definition, we didn't
mention the second input. Our definition isn't point-free _or_ point-ful--it's
somewhere in between:

    aggregate :: Num c => (a -> c) -> [a] -> c
    aggregate f = sum . map f

The first input, the measuring function `f`, is explicitly mentioned. The other
input, the list of containers, is tacitly left out.

What would the same definition look like if both inputs were made explicit?
Let's apply some **eta-abstraction**,
the [opposite of eta-reduction](https://wiki.haskell.org/Eta_conversion) to find out:

    aggregate f = \cs -> (sum . map f) cs

That looks a bit complicated, so let's simplify a bit:

    aggregate f = \cs -> sum ((map f) cs)
    aggregate f = \cs -> sum (map f cs)
    aggregate f cs = sum (map f cs)

Or, if we want to use `$`:

    aggregate f cs = sum $ map f cs

What's that say, in plain English?

> `aggregate` receives a function and a list of containers,
> maps that function over the containers,
> and returns the result of adding up the outputs.

We have a point-ful definition of `aggregate`.
Could we have a point-free one too? Let's eta-reduce!

    aggregate f = sum . map f
    aggregate f = (.) sum (map f)
    aggregate f = (.) sum $ map f
    aggregate f = (.) sum . map $ f
    aggregate = \f -> (.) sum . map $ f
    aggregate = (.) sum . map
    aggregate = (sum .) . map

Well, that's clear enough! How do we even _read_ `(sum .) . map`?

`aggregate` is somehow a combination of `sum` and `map`. We don't have the
words just yet to describe the nature of that combination, but let's have
a closer look.

We can define a combinator to represent the abstraction `(f .) . g`:

    oo :: (c -> d) -> (a -> b -> c) -> a -> b -> d
    oo = \f g -> (f .) . g

With Haskell's [infix backticks](https://wiki.haskell.org/Infix_operator), we can write:

    aggregate = sum `oo` map

But what is this `oo`? Let's eta-abstract:

    oo = \f g -> (f .) . g
    oo = \f g x -> (f .) . g $ x
    oo = \f g x -> (f .) $ g x
    oo = \f g x -> (f .) g x
    oo = \f g x -> f . g x
    oo = \f g x y -> f . g x $ y
    oo = \f g x y -> f $ g x y

or, if you prefer:

    oo = \f g x y -> f (g x y)

In other words, `oo` is another kind of pipe: in its first phase it applies a
function to two inputs and produces a single output; in the second phase,
it applies that output to another function and returns the final result.

The `oo` combinator is ubiquitous. We use it all the time in expressions like this:

    concatMap = (concat .) . map
    mapM      = (sequence .) . map

We might as well get comfy with `(f .) . g`,
and to that end there's a pretty good explanation
[here](http://stackoverflow.com/questions/20279306/what-does-f-g-mean-in-haskell).

But I'm very excited because **Today I Learned** that `oo` already has a much nicer
name. It's a [`blackbird`](https://hackage.haskell.org/package/data-aviary-0.4.0/docs/Data-Aviary-Birds.html),
so named by logician Raymond Smullyan in his classic puzzle book,
[_To Mock a Mockingbird_](https://en.wikipedia.org/wiki/To_Mock_a_Mockingbird):

> Smullyan's exposition takes the form of an imaginary account of
> going into a forest and discussing the unusual "birds" they find there.
> Each species of bird in Smullyan's forest stands for a particular kind
> of combinator appearing in the conventional treatment of combinatory logic.

So, there:

> `aggregate` is a _blackbird_ of `sum` and `map`.

Doesn't that just fill your heart with song? :D
