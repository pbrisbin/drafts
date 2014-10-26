Some generic *runnable* action,

```ruby
class Action
  def call
  end
end
```

## Rules

- We can't invoke `call` unless we're inside another `Action`s `call`
- We can only make something happen by assigning an instance of `Action` to the
  special variable `$main`
- The "Ruby" runtime will take over and invoke `call` on `$main` when you
  execute your program
- This should trigger all your nested calls to actually happen and your program
  to "do something"

*Note*: Lambdas make nice dummy actions

## Abstractions

Compose actions by applying a function to the (eventual) result,

```ruby
class Action
  # f :: a -> b
  # this :: Action(a)
  # returns Action(b)
  def fmap(&f)
    ->() { f.(this.call) }
  end
end
```

Compose actions by chaining them

```ruby
class Action
  # f :: a -> Action(b)
  # this :: Action(a)
  # returns Action(b)
  def bind(&f)
    f.(this.call)
  end
end
```

Assume some system-provided primitives:

```ruby
class OpenFile < IO
  def initialize(filename)
    @filename = filename
  end

  def call
    system_open(@filename)
  end
end

class GetContents < IO
  def initialize(handle)
    @handle = handle
  end

  def call
    system_read(@handle)
  end
end

class PutContents < IO
  def initialize(handle, contents)
    @handle = handle
    @contents = contents
  end

  def call
    system_write(@handle, @contents)
  end
end
```

Wrap them in global functions so we can stay sane:

```ruby
def openFile(filename)
  OpenFile.new(filename)
end

def getContents(handle)
  GetContents.new(handle)
end

def putContents(handle, contents)
  PutContents.new(handle, contents)
end
```

We can use these to write useful programs without calling `call` anywhere
ourselves:

```ruby
def readLine
  openFile($stdin).bind do |h|
    getContents(h).bind do |c|
      ->() { c.split("\n").first }
    end
  end
end
```

This is a "Ruby" port of this Haskell code:

```haskell
readLine = openFile stdin >>= getContents >>= return . head . lines
```

```ruby
def putStrLn(c)
  openFile($stdout).bind do |h|
    putContents(h, c)
  end
end
```

This is a "Ruby" port of this Haskell code:

``haskell
putStrLn = openFile stdout >>= putContents
```

Assign to `$main` as that's the only way to trigger actual side-effects

```ruby
$main = readLine.bind { |c| putStrLn(c) }
```

This is a "Ruby" port of this Haskell code:

```haskell
main = readLine >>= putStrLn
```
