I get asked the following question a lot:

> If Haskell is pure and there are no side-effects, how do you *do* anything?

It's a fair question. The language is pure and there are no side-effects. And
yet, people write [command-line applications][pandoc] and [web
frameworks][yesod] every day. How is this possible?

[pandoc]: http://johnmacfarlane.net/pandoc/
[yesod]: http://www.yesodweb.com/

The short answer is that, when it comes to interaction with the outside word,
Haskell cleanly separates *definition* from *evaluation*. The code you write
provides the former -- it's entirely pure and free of side-effects. The latter,
on the other hand, is controlled entirely by the compiler.

## List of a

The type `[a]` (pronounced *List of a*) is what's known as a *higher-kinded* or
*parameterized* type. It represents a collection of elements of type `a`. Using
`a` here is only convention, we could've used any lower case letter. `[a]` is
one example of a "simple" type that shares many qualities with `IO a`
(pronounced *IO of a*) which Haskell uses to represent interactions with the
outside world.

When defining functions which operate on lists, but don't access the contained
values, you don't need to specify what that `a` is since it doesn't matter:

```haskell
reverse :: [a] -> [a]
reverse = undefined
```

Leaving the elements' type unspecified has a surprisingly limiting effect. The
`reverse` function doesn't *care* what the list contains, but it also doesn't
*know* what the list contains. Therefore, it is impossible for `reverse` to do
anything at all with the list elements themselves. That leaves 3 things this
function could do:

- Return the list
- Return the list, with elements in a different order
- Return the list, with elements missing or duplicated

3 things. **3 things**. Of the infinite possibilities and edge-cases we as
developers have to account for on a daily basis, I showed you a function which
can have only 3 behaviors. Pure functions are amazing.

But I digress.

## IO of a

Let's look at a Haskell function which does nothing. It only describes how to do
something:

```haskell
readFile :: FilePath -> IO String
readFile = undefined
```

This function *describes the act of reading a file*. It does not read the file,
it merely describes how to do it.

## Fmap

What if we wanted to read a file, but get its lines in reverse (i.e. the `tac`
program)? We need a function which reverses the lines of the *eventual* result
from `readFile`. How can we define such a function?

```haskell
tac :: FilePath -> IO String
tac fp = fmap reverseLines $ readFile fp

  where
    -- pure function, reverses the lines in a string
    reverseLines :: String -> String
    reverseLines s = unlines $ reverse $ lines s
```

## Apply

Requirements change. `tac` is out, `diff` is in. Luckily, we've got a pure
function for calculating differences in strings:

```haskell
data Differences = Differences -- not important

diff :: String -> String -> Differences
diff = undefined -- also not important
```

The problem is that the two strings we want to `diff` will come from reading two
files. We can only *describe* reading files, we can't actually read them. How do
we take those two actions and build a new action which, when executed, would
read the two files and return their differences as its result?

If you tried to do this with only `fmap`, you would fail. We can't do this with
the tools we have so far. We need more power.

What if we had a function called `apply` with the following signature:

```
apply :: IO (a -> b) -> IO a -> IO b
apply = undefined
```

It's not immediately clear that this is useful, but let's run with it for a
second. We need some value with the type `IO (a -> b)` to even use the thing. We
can actually get that by mapping `diff` over the first `readFile` action. You
may be confused that we can do this since `diff` takes two arguments, but it's
perfectly fine thanks to *partial-application* and *currying*.

Let's walk through the types:

```haskell
fmap :: (a -> b) -> IO a -> IO b

--      (a      ->  b                     )
diff :: (String -> (String -> Differences))

--           IO a      -> IO  b
fmap diff :: IO String -> IO (String -> Differences)

--                IO a
readFile aFile :: IO String

--                            IO  b
fmap diff (readFile aFile) :: IO (String -> Differences)
```

Now that we have that `IO (a -> b)`, we can give it to `apply` with some other
`IO a`. That's just the action for reading the other file.

```haskell
--     IO (a -> b)                  IO a                     IO b
apply (fmap diff (readFile aFile)) (readFile anotherFile) :: IO Difference
```

That's exactly what we needed! We've constructed an action which, when executed,
will read the two files and present their differences as its result.

About those parenthesis though... Let's use another Haskell trick: if you have a
function that takes two (or more) arguments, you can use *infix notation*. By
surrounding the function in backticks, you can place it *between* its arguments:

```haskell
diff `fmap` readFile aFile `apply` readFile anotherFile
```

That's much better, but we can go a bit further. If a function is entirely
symbolic (e.g. `+`), it can be used as infix without the backticks. This is why
expressions like `2 + 2` can be written in this intuitive way even though `+` is
actually a function, not an operator. Combine this with the fact that Haskell
has symbol-only versions of `fmap` and `apply` and you get:

```haskell
ioDiff :: FilePath -> FilePath -> IO Differences
ioDiff fp1 fp2 = diff <$> readFile fp1 <*> readFile fp2
```

That's almost as compact and easy as just applying `diff` to two strings, except
now we have an action that can eventually be bound to `main` and used to
interact with the outside world.

## Bind

Let's say we wanted to write a function like `cat` which reads a file and prints
its contents. We have the function `readFile` which will produce the file's
contents when executed, and there's another function which describes the act of
printing a value to the terminal screen:

```haskell
putStr :: String -> IO ()
putStr = undefined
```

*TODO: discuss () at all?*

What happens when we `fmap` one action over another action?

```haskell
cat :: FilePath -> IO (IO String)
cat fp = fmap putStr $ readFile fp
```

Uh-oh, `IO (IO a)`? That can't be good. We need a way to collapse the nested
contexts down to just one:

```haskell
flatten :: IO (IO a) -> IO a
flatten = undefined

cat :: FilePath -> IO String
cat fp = flatten $ fmap putStr $ readFile fp
```

Hmm, that's a bit ugly. Let's factor out a helper and again take advantage of
infix notation:

```haskell
flatten :: IO (IO a) -> IO a
flatten = undefined

andThen :: IO a -> (a -> IO b) -> IO b
andThen a f = flatten $ fmap f a

cat :: FilePath -> IO ()
cat fp = readFile fp `andThen` putStr
```

Versions of both `flatten` and `andThen` exist in the standard library. They are
part of the `Monad` type class. The former is called `join` and the latter
`(>>=)`, pronounced *bind*. It's a symbol-only function because it's most useful
in infix notation (as you've seen).

```haskell
cat fp :: FilePath -> IO ()
cat fp = readFile fp >>= putStr
```

When a type implements the `Monad` interface, they're free to define `join` or
`(>>=)` since either can be defined in terms of the other.

## Do-Notation

Haskell allows for expressions to be written in something called
*do-notation*. All that means is you can write something like this:

```haskell
cat fp = do
    contents <- readFile fp

    putStr contents
```

And, before compiling, Haskell will translate it into:

```haskell
cat fp = readFile fp >>= \contents -> putStr contents
```

This notation is a double-edged sword. It makes Haskell code using the I/O Monad
appear very imperative (as is its intent). But for someone new to Haskell,
attempting to gain anything more than a superficial understanding of what this
code is doing is nearly impossible. It looks like magic.

By backing all the way up to the (arguably) simple idea of Typed Effects, then
showing how we can map functions over them with `fmap`, and combine them via
`apply` and `andThen`, we've seen that there is no magic here. All it is are
pure values being passed to functions -- and somehow, we still get things done.

Bringing this all together, you should now know what the following mostly
complete, side-effecting, and somewhat inefficient program does. More
importantly, you should know *how* it does it even in a completely pure
language.

```haskell
main :: IO ()
main = do
    differences <- diff
        <$> readFile "foo.txt"
        <*> readReversed "foo.txt"

    putStr $ show differences

readReversed :: FilePath -> IO String
readReversed fp = fmap reverseLines $ readFile fp

  where
    reverseLines :: String -> String
    reverseLines s = unlines $ reverse $ lines s

diff :: String -> String -> Differences
diff = undefined
```
