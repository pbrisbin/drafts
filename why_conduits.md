I found my Google Drive sync client can have some pretty long downloads or
uploads for larger files. Rather than hanging there waiting, I wanted to output
progress information. At first, only kb/s or something, but I was also thinking
about `curl`-like progress bars.

Naively, you'd expect an API libary to expose something like this:

```haskell
downloadFile :: File -> FilePath -> Api ()
```

This means that, regardless of how it's done, progress reporting has to be baked
into this function -- either as a one-size-fits-all thing or configurable by
additional arguments. Sigh.

Instead, I ended up refactoring the API to provide a lower-level function:

```haskell
type Source = ...
    -- Some specialization of ResumableSource m e

getSource :: URL -> (Source -> Api a) -> Api a
```

On top of this, I can still give the simpler interface if I want:

```haskell
downloadFile :: File -> FilePath -> Api ()
downloadFile file filePath = getSource
    (downloadUrl file) ($$+- sinkFile filePath)
```

But, as library user, I could now implement the progress logic myself:

```haskell
myDownloadFile :: File -> FilePath -> Api ()
myDownloadFile file filePath = getSource
    (downloadUrl file) ($$+- withProgress =$ sinkFile filePath)

withProgress :: MonadIO m => Conduit ByteString m ByteString
withProgress = undefined
    -- v <- await
    -- liftIO $ report progress
    -- yield v
    -- loop
```

I found immediately that this interface scales to other manipulations of the
download behavior. For instance, I noticed that I could easily saturate my
network while the sync ran, causing my computer to get really slow. Using the
same process, I wrote a "throttling" `Conduit` and stuck that in the pipeline:

```haskell
myDownloadFile :: File -> FilePath -> Api ()
myDownloadFile file filePath = getSource
    (downloadUrl file) ($$+- throttled =$ withProgress =$ sinkFile filePath)

throttled :: MonadIO m => Conduit ByteString m ByteString
throttled = undefined
    -- v <- await
    -- liftIO $ check speed, sleep if needed
    -- yield v
    -- loop
```

The best part? These `Conduit`s are also directly reusable for uploads!

```haskell
uploadFile :: FilePath -> File -> Api ()
uploadFile filePath file = do
    request <- uploadRequest file

    performRequest $ setRequestBodySource request $
        withProgress $= throttled $= sourceFile filePath
```

Nicely done Snoyman. Nicely done.
