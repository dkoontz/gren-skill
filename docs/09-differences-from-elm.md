# 09 — Differences from Elm

Gren began as a fork of Elm but has diverged significantly. **Elm code will not compile as Gren.** This doc catalogues the differences that come up most often when helping a user who knows Elm.

> If a user is copying code from an Elm tutorial or package and it is not compiling, this is the doc to load. Walk them through the relevant change below.

## Pattern matching: `when is`, not `case of`

```elm
-- Elm
case maybeValue of
    Just x -> x
    Nothing -> default
```

```gren
-- Gren
when maybeValue is
    Just x -> x
    Nothing -> default
```

Same semantics, new keywords for readability.

## Custom type constructors: 0 or 1 argument, never more

Gren constructors take at most one argument. Wrap multiple values in a record.

```elm
-- Elm
type Person = Person String Int
```

```gren
-- Gren
type Person = Person { name : String, age : Int }
```

This is also why Gren removed tuples — named fields read better than positional ones.

## No tuples

```elm
-- Elm
position : ( Int, Int )
position = ( 10, 20 )
```

```gren
-- Gren
type alias Position = { x : Int, y : Int }
position : Position
position = { x = 10, y = 20 }
```

This also affects `update`'s return type. Elm's `( Model, Cmd Msg )` becomes `{ model : Model, command : Cmd Msg }` (an anonymous record) in Gren. See the sample Node program's `update` at `src/Main.gren` around lines 111–148 for a concrete example.

## Arrays, not Lists

Gren's default sequential data structure is `Array`, an immutable array. There is **no `List`** in the standard library the way Elm has it.

```elm
-- Elm
numbers : List Int
numbers = [ 1, 2, 3 ]
```

```gren
-- Gren
numbers : Array Int
numbers = [ 1, 2, 3 ]
```

The same literal syntax `[ … ]` produces an `Array` in Gren. Code that expects `List` will not compile.

## No automatic record constructors

Elm generates a constructor function from a record type alias:

```elm
-- Elm
type alias Person = { name : String, age : Int }
alice = Person "Alice" 30   -- works
```

Gren does **not**:

```gren
-- Gren
type alias Person = { name : String, age : Int }
alice = { name = "Alice", age = 30 }   -- must use the literal syntax
```

## Renamed collection functions

| Elm (`elm-lang/core`)  | Gren              | Why                                |
| ------------ | ----------------- | ---------------------------------- |
| `filter`     | `keepIf`          | Says what it does                  |
| `filterMap`  | `mapAndKeepJust`  | Describes the two operations       |
| `concatMap`  | `mapAndFlatten`   | Describes the two operations       |
| `Dict.insert`| `Dict.set`        | Same name as `Set.set` — consistent across both |
| `Set.insert` | `Set.set`         | Same name as `Dict.set` — consistent across both |
| `Dict.size`  | `Dict.count`      | Consistent with `Set.count`        |
| `Set.size`   | `Set.count`       | Consistent with `Dict.count`       |
| `Dict.fromList` | (none)         | No built-in helper in 7.4.2; fold manually |
| `Set.fromList`  | `Set.fromArray`| `List` does not exist; the input is an `Array` |
| `Date` module          | (none; `Time` only) | No `Date`/`Date.fromString` in core — parse to an opaque wrapper of `Time.Posix` (see `04-data-structures.md`) |

`length` keeps its Elm name in Gren (applies to `Array.length`, `String.length`). `flatMap` does not exist in `elm-lang/core` (that name is used by some third-party Elm packages); in `elm-lang/core` the function is `concatMap`. If you copy code from Elm and `filter`, `Dict.insert`, or `Date` is undefined, the table above is why.

## No `map4`, `map5`, … — use `andMap`

Gren removed the higher-arity `mapN` functions from `Maybe`, `Result`, `Task`, etc. Keep `map2`/`map3` for small cases; for anything larger, use `andMap`:

```elm
-- Elm
Maybe.map4 buildRecord maybeA maybeB maybeC maybeD
```

```gren
-- Gren
Maybe.map buildRecord
    |> Maybe.andMap maybeA
    |> Maybe.andMap maybeB
    |> Maybe.andMap maybeC
    |> Maybe.andMap maybeD
```

The same pattern applies to `Result.andMap`, `Task.andMap`, etc.

## Record update with any expression

In Elm, the left of `|` in a record update must be a variable:

```elm
-- Elm
updated = { myRecord | value = "new" }
```

Gren allows **any expression** that yields a record:

```gren
-- Gren
updated = { SomeModule.getRecord | value = "new" }
updated = { getConfig env | timeout = 30 }
```

This removes the need for an intermediate `let` binding.

## Extended record pattern matching

Gren extends what `when … is` can match against, including record fields directly:

```gren
describeMixture : { leftQty : Int, rightQty : Int } -> String
describeMixture mixture =
    when mixture is
        { leftQty = 0 } -> "No left ingredient"
        { rightQty = 0 } -> "No right ingredient"
        _ -> "Has both ingredients"
```

This is more flexible than Elm's record pattern matching. Make use of it when modelling rules — it can eliminate `if` chains.

## Unit type is `{}`, not `()`

```elm
-- Elm
doNothing : () -> ()
doNothing _ = ()
```

```gren
-- Gren
doNothing : {} -> {}
doNothing _ = {}
```

`{}` is both the type and the value. It comes up in low-level library signatures and ports.

## Tasks run concurrently by default

In Elm, `Task.attempt` sequences. In Gren, `Task.map2`, `Task.map3`, and `Task.andMap` **run their tasks concurrently** (`gren-lang/core` 7.4.2 docs: *"`map2` starts both requests concurrently and if either of them fails, the whole thing fails!"*). If order matters, use `Task.sequence`:

```gren
-- Concurrent
Task.map2 combineResults
    taskProducingA
    taskProducingB

-- Sequential
Task.sequence [ taskA, taskB, taskC ]

-- Parallel for an array of tasks
Task.concurrent [ taskA, taskB, taskC ]
```

This is the most surprising change for Elm users porting code. See `08-task-ports.md`.

## Task ports

Gren adds task ports as a simpler JS interop model. A single port declaration gives request/response semantics:

```gren
port saveToDB : String -> Task Json.Decode.Value Int
```

With classic Cmd/Sub ports you would write two declarations (an outgoing `Cmd` port plus an incoming `Sub` port) and manage correlation yourself. See `08-task-ports.md`.

## What did NOT change

Plenty of Elm muscle memory still applies in Gren:

- Pure functional architecture, `Model`/`Msg`/`update`/`view`.
- Immutability, no `null`, no exceptions.
- `Maybe`/`Result`, `String`, `Dict`, `Set`.
- Currying, partial application, `|>`.
- The module system (`module … exposing (…)` and `import`).
- Type annotations with `:`, `->`, lowercase type variables.

If a user is generally confused, anchor on what they know and walk them to the Gren form using the table above.

## Quick porting checklist

When converting Elm code to Gren:

1. `case … of` → `when … is`
2. `( a, b )` tuples → record with named fields
3. `List a` → `Array a`
4. Multi-arg constructors → single-arg constructor wrapping a record
5. `filter` → `keepIf`, `Dict.insert`/`Set.insert` → `Dict.set`/`Set.set`, `Dict.size`/`Set.size` → `Dict.count`/`Set.count` (see table above)
6. Higher `mapN` → `map` + `andMap`
7. Record update with expressions — go ahead, it just works
8. `()` → `{}`
9. `update` returning `(model, cmd)` → `{ model = model, command = cmd }`
10. Inspect concurrency — if Elm code relied on `Task.attempt` ordering, switch to `Task.sequence`
11. `Task.perform`/`Task.execute` require `Task Never a` (infallible by type); use `Task.attempt` for fallible work (see `08-task-ports.md`)

## Cross-references

- Where things live now: `01-values-and-functions.md` through `08-task-ports.md`
- Elm equivalents are noted inline throughout those docs.
