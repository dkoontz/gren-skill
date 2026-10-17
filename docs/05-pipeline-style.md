# 05 — Pipeline Style

Gren has a strong idiom: write functions **data-last**, then chain them with the `|>` operator. This is more than a stylistic preference — it shapes how library APIs are designed.

## The `|>` operator

`x |> f` is exactly `f x`. It lets you read a transformation left-to-right, top-to-bottom:

```gren
-- nested calls, read inside-out
result = String.toUpper (String.trim (String.append "hello" " world"))

-- pipelined, read top-to-bottom
result =
    "hello"
        |> String.append " world"
        |> String.trim
        |> String.toUpper
```

Both forms compile to the same thing. The pipelined version is idiomatic once a chain is more than two calls deep.

## Data-last argument order

For `|>` to read naturally, the **data being transformed** must be the last argument to each function:

```gren
Array.map : (a -> b) -> Array a -> Array b
--                ^^^ arg1           ^^^ arg2 (data) — last
```

That way, you can write:

```gren
numbers
    |> Array.map (\n -> n * 2)
    |> Array.keepIf (\n -> n > 5)
    |> Array.foldl (+) 0
```

If the data were the first argument, you could not pipe between these calls. This is why every well-designed Gren function puts configuration/options first and the principal data last.

### When you have to put data first

Sometimes a function naturally takes data first (a record field accessor, a getter). You can still use it in a pipeline via a lambda or by partial application:

```gren
user |> .name

config |> Maybe.map .timeout
```

The `.field` accessor is itself data-last when written as `.field`, which makes it pipe-friendly.

## Multi-step transformation example

From `src/Main.gren` around lines 117–123:

```gren
parsed =
    line
        |> String.trim
        |> String.toLower
        |> parseCommand
```

Reading it aloud: "take the line, trim it, lowercase it, parse it." This is the natural shape of input-processing code in Gren.

## Combining with `Maybe` and `Result`

The pipeline style extends naturally to error-handling types:

```gren
userName : Maybe User -> String
userName maybeUser =
    maybeUser
        |> Maybe.map .name
        |> Maybe.withDefault "Unknown"
```

And to `Result`:

```gren
portNumber : String -> Result String Int
portNumber input =
    String.toInt input
        |> Result.fromMaybe "Invalid number"
        |> Result.andThen
            (\n ->
                if n > 0 && n < 65536 then
                    Ok n
                else
                    Err "Port must be between 1 and 65535"
            )
```

`Result.fromMaybe` attaches an error to a `Maybe` (the Maybe becomes `Ok`/`Err`); `Result.andThen` threads the fallible steps.

## Building pipelines from existing functions

A pipeline step can be:

- A function name: `String.trim`
- A partially applied function: `Array.map (\n -> n * 2)` or `Dict.set "key" 42`
- A field accessor: `.name`
- A lambda: `\x -> x * x`

If you find yourself writing a long lambda inline, factor it into a named helper — the pipeline stays readable and the helper gets a useful type signature.

## The reverse pipeline `<|`

`f <| x` is `f x`. It exists but is rarely idiomatic. The common legitimate use is to avoid a wrapping pair of parentheses in a single function call:

```gren
respond model (String.join "\n" lines)
-- vs
respond model <| String.join "\n" lines
```

Use sparingly. `|>` is the dominant direction.

## Function composition `>>` and `<<`

`f >> g` is `\x -> x |> f |> g`. `<<` is the reverse. These are useful when building a transformation to apply later, not when you already have the data:

```gren
normalise : String -> String
normalise =
    String.trim >> String.toLower
```

You will see composition far less than pipelines in everyday Gren.

## Why this matters for coaching

When a user asks "how do I do X with this data", the idiomatic Gren answer is almost always a pipeline. Coach them to:

1. Identify the steps in order.
2. Look for the data-last function for each step (and discover its name from `gren-lang/core` if unsure).
3. Compose them with `|>`.

If a step has no clean data-last function, that is often a signal that the data is shaped wrong (a custom type might help — see `04-data-structures.md`).

## Cross-references

- Data-last functions are most often found on `Array`, `Dict`, `Maybe`, `Result`: `04-data-structures.md`, `06-errors.md`
- `when … is` is how you branch *inside* a pipeline step: `03-decisions.md`
- Example pipelines in real code: `src/Main.gren` around lines 117–123 and 172–180
