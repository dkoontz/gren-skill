# 06 — Errors and Optional Values

Gren has **no null, no exceptions, and no `try`/`catch`**. Absence and failure are values, represented by `Maybe` and `Result`. The type system forces every callsite to deal with them.

## The `Maybe` type

`Maybe a` represents "value that may be absent".

```gren
type Maybe a
    = Nothing
    | Just a
```

### When to use `Maybe`

- A lookup may not find anything (`Dict.get`, `Array.get`).
- A parse may not succeed (`String.toInt`).
- A field is genuinely optional in your domain.

If the failure has a **reason** (parse error message, HTTP status, business rule), prefer `Result` instead — see below.

### Consuming `Maybe`

Always pattern-match:

```gren
userName : Maybe User -> String
userName maybeUser =
    when maybeUser is
        Just user -> user.name
        Nothing -> "Unknown"
```

Or use the helper functions:

```gren
userName2 : Maybe User -> String
userName2 maybeUser =
    maybeUser
        |> Maybe.map .name
        |> Maybe.withDefault "Unknown"
```

Common `Maybe` helpers (`gren-lang/core` 7.4.2):

| Function                  | Type                                       | Purpose                                       |
| ------------------------- | ------------------------------------------ | --------------------------------------------- |
| `Maybe.map`               | `(a -> b) -> Maybe a -> Maybe b`           | Transform a present value                     |
| `Maybe.andThen`           | `(a -> Maybe b) -> Maybe a -> Maybe b`     | Chain fallible steps                          |
| `Maybe.withDefault`       | `a -> Maybe a -> a`                        | Extract with a fallback                       |
| `Maybe.withDefaultLazy`   | `({} -> a) -> Maybe a -> a`                | Extract with a lazily-computed fallback (the `{}` argument is Gren's unit value) |
| `Maybe.keepIf`            | `(a -> Bool) -> Maybe a -> Maybe a`        | `Nothing` if the value fails the test         |
| `Maybe.map2`              | `(a -> b -> c) -> Maybe a -> Maybe b -> Maybe c`         | Combine two `Maybe`s, or `Nothing` if either is absent |
| `Maybe.map3`              | `(a -> b -> c -> d) -> Maybe a -> Maybe b -> Maybe c -> Maybe d` | Combine three `Maybe`s |
| `Maybe.andMap`            | `Maybe a -> Maybe (a -> b) -> Maybe b`     | Combine N maybes; the idiomatic way past `map3` (see below) |

### Combining multiple `Maybe`s

Gren's core modules define `mapN` only up to arity 3. To combine more than three, start from `map` and pipe through `andMap`:

```gren
-- not valid Gren
Maybe.map4 buildRecord maybeA maybeB maybeC maybeD

-- valid
Maybe.map buildRecord
    |> Maybe.andMap maybeA
    |> Maybe.andMap maybeB
    |> Maybe.andMap maybeC
    |> Maybe.andMap maybeD
```

`map2`/`map3` still exist for small cases.

## The `Result` type

`Result e a` represents "a computation that fails with a typed error".

```gren
type Result e a
    = Err e
    | Ok a
```

The error type `e` is **whatever you choose** — usually a custom type so callers can pattern-match on specific failure modes.

### When to use `Result`

- Parsing typed input (number out of range, malformed shape).
- I/O that can fail (HTTP, file reads).
- Business validation with a meaningful error.

Rule of thumb: if you would otherwise attach a string saying *why* a `Maybe` was `Nothing`, switch to `Result`.

### Example from the project

`src/Main.gren` around lines 290–322:

```gren
parseCommand : String -> Result InvalidCommand Command
parseCommand input =
    if input == "help" then
        Ok Help

    else if input == "quit" || input == "exit" then
        Ok Quit

    -- ... more branches ...

    else
        Err (UnknownCommand input)
```

And the custom error type, around lines 48–53:

```gren
type InvalidCommand
    = UnknownCommand String
    | InvalidDirection String
    | InvalidItem String
    | ItemNotHere String
```

Notice each variant carries a `String` — typically the offending input — so `formatError` can produce a useful message. The caller (`update`, around lines 114–130) pattern-matches on `Ok`/`Err` and never has to guess whether parsing succeeded.

### Consuming `Result`

```gren
when parseCommand input is
    Ok cmd -> handleCommand model cmd
    Err err -> respond model [ formatError err ]
```

Or with helpers (`gren-lang/core` 7.4.2):

| Function                | Type                                            | Purpose                                       |
| ----------------------- | ----------------------------------------------- | --------------------------------------------- |
| `Result.map`            | `(a -> b) -> Result e a -> Result e b`          | Transform a success value                     |
| `Result.map2`           | `(a -> b -> c) -> Result x a -> Result x b -> Result x c`            | Combine two results; first `Err` propagates   |
| `Result.map3`           | `(a -> b -> c -> d) -> Result x a -> Result x b -> Result x c -> Result x d` | Combine three results |
| `Result.andMap`         | `Result x a -> Result x (a -> b) -> Result x b` | Combine N results; the idiomatic way past `map3` |
| `Result.mapError`       | `(e -> f) -> Result e a -> Result f a`          | Transform the error value                     |
| `Result.andThen`        | `(a -> Result e b) -> Result e a -> Result e b` | Chain fallible steps                          |
| `Result.onError`        | `(e -> Result f a) -> Result e a -> Result f a` | Recover from an error mid-chain               |
| `Result.withDefault`    | `a -> Result e a -> a`                          | Extract with a fallback                       |
| `Result.withDefaultLazy`| `({} -> a) -> Result e a -> a`                  | Extract with a lazily-computed fallback       |
| `Result.toMaybe`        | `Result e a -> Maybe a`                         | Discard the error                             |
| `Result.fromMaybe`      | `e -> Maybe a -> Result e a`                    | Attach an error to a maybe                    |
| `Result.firstOk`        | `Array (Result x a) -> Maybe a`                 | First `Ok` in an array                        |
| `Result.allOk`          | `Array (Result e a) -> Result (Array e) (Array a)` | All-or-errors collection                   |

### Chaining `Result` steps

`Result.andThen` is the key tool — it threads fallible computations together:

```gren
parsePort : String -> Result String Int
parsePort input =
    String.toInt input
        |> Result.fromMaybe "Invalid number"
        |> Result.andThen
            (\port ->
                if port > 0 && port < 65536 then
                    Ok port
                else
                    Err "Port must be between 1 and 65535"
            )
```

The moment any step returns `Err`, the rest of the chain is skipped and the error propagates. This is the closest Gren gets to "exceptions", and it is fully typed.

## Custom error types — the recommended pattern

1. Define an error type as a custom type with one variant per failure mode:

   ```gren
   type SaveError
       = DiskFull
       | PermissionDenied FilePath
       | NetworkTimeout
   ```

2. Return `Result SaveError Something` from the operation.

3. At the boundary where you surface the error to the user (or log it, or render it), pattern-match on the variants and format a useful message.

This beats returning `Result String a` for anything non-trivial — strings are easy to write but hard for callers to act on programmatically.

## What about side-effecting failures?

Real side effects (HTTP, file I/O, network) do not happen inside ordinary Gren functions. They happen in **commands** (`Cmd`) and **tasks** (`Task`), and the runtime hands the result back to your `update` function wrapped in a `Msg` carrying `Result`. See `07-commands-and-subs.md` and `08-task-ports.md`.

So the typical shape is:

```gren
type Msg
    = GotResponse (Result Http.Error String)

update : Msg -> Model -> { model : Model, command : Cmd Msg }
update msg model =
    when msg is
        GotResponse result ->
            when result is
                Ok data ->
                    { model = { model | data = data }
                    , command = Cmd.none
                    }

                Err error ->
                    { model = { model | error = Just error }
                    , command = Cmd.none
                    }
```

Note the anonymous-record return shape — `update` returns `{ model = …, command = … }`; Gren has no tuples, so multi-value results always travel in named fields (see `02-type-signatures.md`).

## Quick decision guide

| Situation                                                | Use                     |
| -------------------------------------------------------- | ----------------------- |
| Optional field or lookup, no meaningful failure reason    | `Maybe`                 |
| Parsing, validation, or an operation with a typed reason | `Result` with custom err|
| Side-effecting I/O                                       | `Task`/`Cmd` returning `Result` |
| "Just crash" — there is no such thing                    | — design out the case   |

## Cross-references

- Pattern matching on these types: `03-decisions.md`
- The custom type machinery: `04-data-structures.md`
- Where errors from I/O actually live: `07-commands-and-subs.md`, `08-task-ports.md`
- The Elm equivalents of everything renamed or removed above: `09-differences-from-elm.md`
