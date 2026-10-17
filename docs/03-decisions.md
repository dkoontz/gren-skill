# 03 — Decisions

How to branch in Gren: `if`/`then`/`else` for binary choices and `when … is` for everything richer. Both are **expressions** — they return values.

## `if` expressions

Form: `if cond then exprA else exprB`. The `else` branch is **mandatory**; there is no `if` without `else`, because every expression must return a value.

```gren
sign : Int -> String
sign n =
    if n < 0 then
        "negative"

    else if n == 0 then
        "zero"

    else
        "positive"
```

The condition must be a `Bool`. There are no truthy/falsy values — `0`, `""`, and `{}` are all distinct from `False`.

## `when … is` (pattern matching)

Gren renamed Elm's `case … of` to `when … is`. Same semantics, different keywords:

```gren
-- Elm (does NOT work in Gren)
case maybeValue of
    Just x -> x
    Nothing -> default

-- Gren
when maybeValue is
    Just x -> x
    Nothing -> default
```

`when` is the workhorse of Gren code. It branches on **patterns**, not just booleans. Each branch is `pattern -> expression`. Indentation lines up the arrows.

### Exhaustiveness

`when` must cover every possible pattern for the type. The compiler will reject a `when` on a custom type if any constructor is missing. For strings, numbers, etc. you provide a `_` (wildcard) catch-all:

```gren
parseDirection : String -> Maybe Direction
parseDirection str =
    when str is
        "north" -> Just North
        "south" -> Just South
        "east"  -> Just East
        "west"  -> Just West
        _       -> Nothing
```

Real example from `src/Main.gren` around lines 324–340.

### Literal patterns

You can match on `Int`, `Float`, `Char`, `String`, and `Bool` literals:

```gren
describe : Int -> String
describe n =
    when n is
        0 -> "zero"
        1 -> "one"
        _ -> "many"
```

### Variable (binding) patterns

A lowercase identifier in a pattern **binds** that part of the value to a name:

```gren
identity : Maybe a -> Maybe a
identity m =
    when m is
        Just x -> Just x   -- x is bound to the inner value
        Nothing -> Nothing
```

### Custom type patterns

Match each constructor. If a constructor carries data, you can bind it positionally (Gren custom type constructors take at most one argument — see `04-data-structures.md` § Custom types):

```gren
describeState : ConnectionState -> String
describeState state =
    when state is
        Disconnected -> "Not connected"
        Connecting -> "Attempting to connect..."
        Connected { sessionId } -> "Connected: " ++ SessionId.toString sessionId
        Failed { error } -> "Failed: " ++ ConnectionError.toString error
```

Notice the inner `{ sessionId }` record pattern — when the single constructor argument is a record, you can destructure its fields directly.

### Record patterns

You can match on record fields directly:

```gren
describeMixture : { leftQty : Int, rightQty : Int } -> String
describeMixture mixture =
    when mixture is
        { leftQty = 0 } -> "No left ingredient"
        { rightQty = 0 } -> "No right ingredient"
        _ -> "Has both ingredients"
```

Gren extends record pattern matching beyond what Elm offers — see `09-differences-from-elm.md`.

### Wildcard `_`

Matches anything and binds nothing. Use it for catch-alls or to ignore a value:

```gren
alwaysZero : a -> Int
alwaysZero _ = 0
```

### Nested patterns

Patterns compose. You can match a `Maybe` inside a custom type inside a record:

```gren
when result is
    Ok { user = Just u } -> greet u
    Ok { user = Nothing } -> "no user"
    Err e -> "error: " ++ e
```

## No guards

Gren does **not** support `| predicate ->` guards inside `when` (Elm does not either). If you want a conditional branch, use `if` inside the branch body, or restructure the data so the predicate becomes a pattern.

```gren
-- not valid
when n is
    x | x > 0 -> "positive"
    _ -> "non-positive"

-- do this instead
when n > 0 is
    True -> "positive"
    False -> "non-positive"

-- or pattern-match on a custom-shaped value
```

## Choosing between `if` and `when`

- **`if`** is for boolean conditions, especially when there is nothing to destructure.
- **`when`** is for choosing among variants, especially custom types, `Maybe`/`Result`, and literal patterns. It scales to many branches without nesting.
- Deeply nested `if … then … else if …` chains are usually clearer as a `when` over a custom type. If you find yourself writing three or more branches as `else if`, consider redesigning the data.

In `src/Main.gren`, `parseCommand` (around lines 290–322) uses a long `if/else if` chain on `String.startsWith`. That works, but a coach might discuss whether modelling the verb as data and pattern matching would read better — without prescribing the change.

## Pattern matching on the result of a function call

You can pattern match on any expression, not just variables:

```gren
when Dict.get id users is
    Just user -> Ok user
    Nothing -> Err NotFound
```

This is idiomatic when you do not need to name the intermediate value.

## Cross-references

- Data shapes you can match against: `04-data-structures.md`
- `Maybe`/`Result` semantics: `06-errors.md`
- The `case of` → `when is` rename from Elm: `09-differences-from-elm.md`
