---
layout: post
title: Writing a redis clone in Haskell for some goddamn reason
---

I've played around a little bit with Haskell, but the closest I've got to doing
something serious with it is writing [a relational algebra
interpreter](https://github.com/culturedsys/kin) (which I did use to do some
homework for a databases class, so I guess that's genuine practical success). I
thought it would be interesting to try and write something practical, and for
some reason I decided to write a [redis](https://redis.io/) clone. On the one
hand, redis provides data structures and various operations on them, we seems
right in Haskell's wheelhouse. On the other, redis exposes these data structures
over a network connection, which is precisely the sort of messy IO Haskell
sweeps under a carpet (or monad). So it seems like an interesting option to
learn how Haskell might fare in the real world. <!-- more -->

## A key-value store

Starting with the more obviously Haskelly part of redis, the key-value store.
redis uses plain ASCII strings as keys:

```haskell
type Key = ByteString
```

Redis stores various different data structures, so values need to be a sum type:

```haskell
data Value = StringValue ByteString | IntValue Int

newtype Store = Store (Map Key Value)
```

That's enough for an initial implementation that supports basic get/put and
increment/decrement operations. In redis, get and put operations always work in
terms of strings, so we need to be able to put strings, and retrieve strings
(converted from ints, if necessary):

```haskell
lookupString :: Store -> Key -> Maybe ByteString
lookupString (Store m) k = case lookup k m of
  Nothing -> Nothing
  Just (StringValue s) -> Just s
  Just (IntValue i) -> Just $ intToByteString i

insertString :: Store -> Key -> ByteString -> Store
insertString (Store m) k s = Store $ insert k (StringValue s) m
```

Increment and decrement, on the other hand, work on integers, converting a
string to an integer if necessary. As not all strings represent integers, the
increment and decrement operations are fallible; they need to be able to return
an error if the string can't be converted to an integer.

```haskell
data Error = BadType
  deriving (Eq, Show)

type Result a = Either Error a

lookupInt :: Store -> Key -> Result (Maybe Int)
lookupInt (Store m) k = case lookup k m of
  Nothing -> Right Nothing
  Just (IntValue i) -> Right i
  Just (StringValue s) -> case byteStringToInt s of
    Nothing -> Left BadType
    Just i -> Right i

insertInt :: Store -> Key -> Int -> Store
insertInt (Store m) k i = Store $ insert k (IntValue i) m

incr :: Store -> Key -> (Store, Result Int)
incr store@(Store m) k = case lookupInt k m of
  Left e -> (store, Left e)
  Right value ->
    let res = (fromMaybe 0 value) + 1
      in (insertInt store k res, Right res)

decr :: Store -> Key -> (Store, Result Int)
decr store@(Store m) k = case lookupInt k m of
  Left e -> (store, Left e)
  Right value ->
    let res = (fromMaybe 0 value) - 1
      in (insertInt store k res, Right res)
```

Another feature of redis, however, calls for a rethinking of this model. Redis
allows setting an expiration time for a key; if the expiration time has passed,
looking up the key acts as if it has never been set. This makes lookup more
complicated in two respects: lookup needs to have access to the current time,
and lookup, which up to know has not modified the store, may now need to remove
a key if the expiry time has passed. We can modify the `Store` easily enough to
add the expiry time as some metadata:

```haskell
data Entry = {
  value :: Value,
  expiry :: Maybe UTCTime
}

newtype Store = Map Key Entry
```

More complicated is modifying the operations to handle expiring entries. This
could be implemented by always passing down the current time to every operation,
and always returning a (potentially) modified store from each operation. To my
mind, though, this is not an optimal solution, because the implementation of the
operation doesn't need to know about the current time (it just needs to pass it
down to the lookup); and a (conceptually) non-modifying operation like `get`
shouldn't have to worry about whether or not the lookup will modify the store.

Luckily, Haskell has a way to handle these kinds of irrelevant details: monads.
When implementing the `get` operation, we want to think in terms of function
with the signature `Key -> Value`, but we need to add something else to the
signature to handle the details which, from the point of view of the function
itself, are irrelevant (the current time and the potential state modification).
Rather than exposing these details directly in the function signature (`UTCTime
-> Key -> (Value, Store)`), we can encapsulate them in a monad. Actually, we
need two monads, `Reader` (which supplies a value), and `State` (which
encapsulates state modification), which we can combine like:

```haskell
type StoreM = ReaderT UTCTime (State Store)
```

Then we can write `get` in a way that is mostly indifferent to the monad:

```haskell
get :: Key -> StoreM (Maybe ByteString)
get k = do
  maybeValue <- lookupValue k
  return $ fmap maybeValue (\case
    IntValue i -> intToByteString i
    StringValue s -> s
  )
```

`lookupValue`, on the other hand, has to be aware of the details within the
monad in order to handle expiry:

```haskell
lookupValue :: Key -> StoreM (Maybe Value)
lookupValue k = do
  (Store m) <- get -- access the value in the `State` monad
  now <- ask -- access the value in the `Reader` monad
  case lookup k m of
    Nothing -> Nothing
    Just entry -> case expiry entry of
      Nothing -> return $ Just (value entry)
      Just expiryTime 
        | now < expiryTime -> return $ Just (value e)
        | otherwise -> do
            -- update the value in the `State` monad 
            -- to remove the expired entry
            put $ Store (M.delete k m)  
            return Nothing
```

The other end of the process, where the operations on the store are run, also
needs to be aware of the monadic details, passing in the current time and
retrieving the updated store, which can be done with:

```haskell
runStoreM :: Store -> UTCTime -> StoreM a -> (a, Store)
runStoreM store now m = runState (runReaderT m now) store
```

There are some additional complications for the updating functions, which have
to deal with either clearing the expiry time (for the normal `set` operation),
setting a new expiry time (for the set with expiration operation), or retaining
the current expiry time (for `incr` and `decr`), but I'll gloss over these.

## A network protocol

Redis exposes this key-value store over the network; the first part of this, the
network protocol, plays to another of Haskell's strengths: it's a parser. Even
better, it's two parsers, one which parses a stream of bytes into general data
structures (strings, ints, arrays, etc), and one which parses the resulting
arrays into redis commands. Redis uses a text-based protocol called
[RESP](https://redis.io/docs/reference/protocol-spec/), and a good choice for
parsing this kind of thing is the
[attoparsec](https://hackage.haskell.org/package/attoparsec) library, which is
intended to be an efficient parser for byte-oriented formats. The RESP protocol
is fairly straightforward, consisting of a small number of types:

```haskell
data Resp
  = SimpleString ByteString
  | Error ByteString
  | Integer Int
  | BulkString ByteString
  | Array [Resp]
  | NullString
```

To distinguish between types, each value starts with a single byte representing
the type (a sigil), followed by the data for the value. A parser for this general format
can be represented as:

```haskell
valueParser :: (a -> Resp) -> Word8 -> Parser a -> Parser Resp
valueParser constructor sigil parser = constructor <$> (word8 sigil *> p)
```

That is, it matches a single byte, which must be the sigil value, ignores it 
with `*>`, matches the supplied parser, and then maps the supplied constructor
over the result.

The `SimpleString` type is represented by a `+` followed by a CRLF terminated
string:

```haskell
crlf = string "\r\n"

crlfTerminated = takeTill (== _cr)

simpleStringParser = valueParser SimpleString _plus crlfTerminated <* crlf
```

The `Integer` type is similar, but starts with a `$` and is followed by a crlf
terminated string of digits.

```haskell
magnitudeParser :: Parser Int
magnitudeParser = fromJust . word8ArrayToInt <$> many1 (satisfy isDigit)

numberParser :: Parser Int
numberParser = do
  sign <- optional (word8 _hyphen)
  magnitude <- magnitudeParser
  return $ if isJust sign then -magnitude else magnitude

integerParser :: Parser Resp
integerParser = valueParser Integer _colon numberParser <* crlf
```

A `BulkString` is signified by a `$` and, unlike a `SimpleString`, includes the
length of the string before the actual string value (this means it can be read
more efficiently, all at once, rather than having to check for a delimiter
character by character). This makes the parser slightly more complicated, but it
can re-use the `magnitudeParser` for reading the length:

```haskell
bulkStringParser :: Parser Resp
bulkStringParser =
  valueParser
    BulkString
    _dollar
    ( do
        len <- magnitudeParser <* crlf
        P.take len <* crlf
    )
```

The RESP protocol also has the concept of a `NullString`, used, like NULL, to
indicate that there is not available value. This is represented using the same
format as a `BulkString`, but with a length of `-1`; as such, it can be parsed
with:

```haskell
nullStringParser :: Parser Resp
nullStringParser =
  valueParser
    (const NullString)
    _dollar
    (word8 _hyphen >> word8 _1 >> crlf)
```

(This means there are two parsers which attempt to consume a string starting
with `$`; however, the `attoparsec` library will backtrack if it encounters,
or doesn't encounter the minus sign in `-1`, so will choose the correct parser
on that basis).

An `Array` works similarly, prefixed with a count of elements; but, whereas each
element of a `BulkString` is a character, each element of an `Array` is another
RESP value. Luckily, Haskell's lazy nature makes recursive parsers easy to write:

```haskell
arrayParser :: Parser Resp
arrayParser =
  valueParser
    Array
    _asterisk
    ( do
        len <- magnitudeParser <* crlf
        count len respParser
    )
```

`respParser` here is the parser for any RESP value, which we haven't implemented
yet, but can now do so, simply as a choice between the different parsers we have 
implemented:

```haskell
respParser :: Parser Resp
respParser =
  simpleStringParser
    <|> errorParser
    <|> integerParser
    <|> bulkStringParser
    <|> nullStringParser
    <|> arrayParser
```

All the redis commands are represented as RESP arrays of strings, which, for
ease of processing can be converted to a list of ByteStrings:

```haskell
extract :: Resp -> Maybe [ByteString] 
extract (Array parts) = traverse (\case BulkString s -> Just s; _ -> Nothing) parts
extract _ = Nothing
```

(I'm once again reminded of how convenient it is to have `traverse` convert a
list of things to a thing of a list, and [sad it's not in available in more
languages](/2021/02/13/Infectious-functional-programming/).)

All redis commands start with the command name, and in most cases this is
followed by a fixed number of arguments with the meaning determined by their
position, as in `GET key`, `SET key value`, etc. These are easy to parse with
pattern matching. The `SET` command also has some additional possible arguments,
like `NX` to specify no overwrite, or `EX n` to set an expiry time, and these
can appear anywhere in the argument list, so can't be matched with a simple
pattern. I have cleverly dealt with these parameters by not implementing them,
which is not as much as a limitation as it might be, as redis has separate
(although deprecated) commands which are easier to pass (`SETNX` and `SETEX`
respectively). So my limited parser can be implemented as:

```haskell
parse :: Resp -> Either CommandError Command
parse resp = case extract resp of
  Just ["SET", k, v] -> Right . Set $ SetArgs k v
  Just ["GET", k] -> Right . Get $ GetArgs k
  Just ["SETNX", k, v] -> Right . SetNx $ SetArgs k v
  Just ["SETEX", k, d, v] -> case byteStringToInt d of
    Nothing -> Left $ ArgParseError d "number"
    Just i ->
      let duration = secondsToNominalDiffTime $ fromIntegral i
      in Right . SetEx $ SetExArgs k duration v
  Just ["INCR", k] -> Right . Incr $ IncrArgs k
  Just ["DECR", k] -> Right . Decr $ DecrArgs k
  _ -> Left CommandParseError
```

## A server

So, now we have a data store and a parser to read commands to execute against 
this store. All we need for a complete redis is to expose this protocol over the 
network. This seems like the part that would be trickiest for Haskell, as it's 
inherently about I/O and side effects, rather than being pure. Even here, 
however, we can largely describe the problem in pure terms. A redis server reads 
a stream of bytes, parses them to produce a stream of commands, applies these 
commands to the data store to produce a stream of results, serializes these 
results to a stream of bytes, and writes them out. This kind of looks like 
composing functions and mapping them over a stream, except for the reading and 
writing and modifying the store in response to commands.[^1] 

Haskell has a number of libraries for expressing side-effecting programs by 
composing operations on streams; I chose to use 
[Conduit](https://hackage.haskell.org/package/conduit), in part because it has 
two convenient facilites: one for setting up a TCP server and then treating each 
connection as a series of stream operations, and another for applying 
`attoparsec` parsers as stream operations. Using this functionality, the server 
looks like:

```haskell
runTCPServer
    (serverSettings 6379 "*")
    ( \app ->
        runConduit $
          appSource app
            .| respParse
            .| C.mapM (processCommand appState)
            .| respWrite
            .| appSink app
    )
```

`runTCPServer` does the main work, setting up a server that listens for 
connections. When a connection is made, it calls the supplied function with a 
parameter containing two streams, one for input and another for output. All my 
program needs to do is join these streams together with a pipeline that parses 
input (`respParse`), executes commands (mapping the `processCommand` function 
over the stream), serialising the result (`respWrite`).

[^1]: The real redis also includes commands to subscribe to a channel, receiving 
notifications when messages are published to that channel. This doesn't fit the 
request/response pattern, so would need to be handled in a more complicated way, 
possibly similar to how this 
[async chat server](https://www.schoolofhaskell.com/user/joehillen/building-an-async-chat-server-with-conduit) 
works.

## Conclusion

While this only implements a tiny fraction of redis functionality, it is enough 
to run the `redis-benchmark` tool for those commands it supports:

| Command | Requests/second |
| ------- | --------------- |
| GET     | 30432           |
| SET     | 29481           |
| INCR    | 16770           |

Running against a real redis server on the same machine gives the following 
benchmarks:

| Command | Requests/second |
| ------- | --------------- |
| GET     | 104712          |
| SET     | 103626          |
| INCR    | 104493          |

So my tiny haskell redis clone runs at about a third of the speed of real redis 
(for GET and SET; about a 6th of the speed for INCR, perhaps because INCR has to 
both get and set). While I would be hard pressed to call this "good" 
performance, I'm still kind of pleased that an unoptimised haskell 
implementation isn't worse. If you're equally easily impressed, the [full source 
code is on GitHub](https://github.com/culturedsys/hedis)
