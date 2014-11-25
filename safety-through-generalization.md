Assuming the author of some function had good intentions, how likely is it to
"get it wrong" but still type-check? Is it possible to reduce this probability
further through good practices?

Such a probability grows as the function becomes more specific in its types. The
more polymorphic the function, the more likely it is that there is only one
possible implementation that type-checks. This is counter-intuitive. You might
think that the more polymorphic a function, the more places it can be used, the
more room there is for error. At least in Haskell, the situation is the
opposite.

As an example, take the function `const`. It takes two arguments and always
returns the first:

```haskell
const :: a -> b -> a
const = undefined

const 1 "foo"
-- => 1
```

I'm using such a simple function because it's easy to wrap your head around, but
don't think this is a contrived example. I assure you, this function is an
incredibly useful building block when writing Haskell. Its correct operation is
important. It's also perfect for this example because it's nearly impossible to
implement it incorrectly! I can't think of any good-faith way (i.e. excluding
`error` or `unsafeCoerce`) to implement this function incorrectly.

The reason this is possible is the generalness of the type signature. This
function must work *for all `a` and `b`*. The only information available to this
function for how to produce a value of type `a` (which it must) is in its first
argument. All it can do is return it as-is, which is exactly how the function is
specified (informally) to behave.

To see how the probability for mistakes increases as we make types more
specific, let's take `map`:

```haskell
map :: (a -> b) -> [a] -> [b]
map = undefined

map (const 1) "Hello"
-- => [1,1,1,1,1]
```

There are indeed ways to get this function wrong. They all stem from the fact
that we've made the function act concretely on lists. It must work *for all `a`
and `b`*, but specifically for lists. That means we could:

- Return the empty list
- Reverse or otherwise reorder the list (while applying the function)
- Return a list of different length by dropping or repeating elements (while
  applying the function)

Even so, all of these mistakes would be hard to make if you had good intentions.
The types have immediately ruled out a whole host of other errors. For example,
we can't *only* reverse the list or *only* change the length because that would
be `[a]` and we're required to return `[b]`. We must apply the function somehow
as that's the only way we have for producing anything of type `b`. Similarly we
can't apply the function only to some elements of the list since a list must be
heterogeneous. Since we've said the function *can* change the type from `a` to
`b` we have to implement `map` assuming that it *will*.

If you were to write a function that relied on `map`, you would by extension
inherit these potential failures and then some:

```haskell
mapTwice :: (a -> a) -> [a] -> [a]
mapTwice f = map f . map f

mapTwice (+2) [1,2,3]
-- => [5,6,7]
```

Now you have all the same potential for mistake, with the addition of forgetting
to apply the function at all or applying it too many times (since it is now `(a
-> a)`, doing so would still type-check). Is there a safer way to do this?

```haskell
mapTwice :: Functor f => (a -> a) -> f a -> f a
mapTwice f = fmap f . fmap f

mapTwice (+2) [1,2,3]
-- => [5,6,7]

mapTwice (+2) (Just 1)
-- => Just 5
```

Not only do you have a more generally useful function, you also gain safety by
specifying that you must work with any `Functor f` and not specifically lists.
This means you can no longer return `[]`, or make a mistake like accidentally
reordering, shortening, or lengthening your result. At this point, the *only*
possible error is applying the function the wrong number of times -- and that's
probably fairly avoidable.
