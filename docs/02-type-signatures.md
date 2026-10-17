# 02 — Type Signatures

Gren is statically typed with full type inference. Type annotations are **optional but recommended** at module boundaries (top-level functions and values). This doc covers how to read and write them.

## The annotation form

A type annotation is `name : Type`, on its own line, directly above the definition:

```gren
name : String
name = "Chorus"

add : Int -> Int -> Int
add x y = x + y
```

The colon is `:`. A common beginner mistake is to confuse this with the definition line, which uses `=`.

## Basic types

| Type      | Literal examples                  | Notes                                                          |
| --------- | --------------------------------- | -------------------------------------------------------------- |
| `Int`     | `0`, `42`, `-7`, `0xFF`             | 32-bit two's-complement semantics in the well-defined range `-2^31` to `2^31 - 1`. Outside that range, behaviour is target-dependent: when compiling to JavaScript the safe range widens to `-(2^53 - 1)` to `2^53 - 1` for some operations (see `Basics.Int`'s note in `gren-lang/core`). |
| `Float`   | `3.14`, `-0.5`                     | IEEE 754                                                       |
| `Bool`    | `True`, `False`                    | Constructors are capitalised                                   |
| `String`  | `"hello"`, `"""multi\nline"""`    | UTF-8 text, immutable                                          |
| `Char`    | `'a'`, `'\n'`                      | Single Unicode scalar value                                    |
| `Bytes`   | (constructed via libraries)        | Raw byte sequences, used by streams and I/O                    |

There is no numeric tower; choose `Int` or `Float` deliberately.

## Function types

A function type is a chain of arrows. Arrows are **right-associative**:

```gren
add : Int -> Int -> Int
-- parsed as: Int -> (Int -> Int)
```

So `add 1` produces an `Int -> Int`. This is currying — see `01-values-and-functions.md` § Partial application.

When a function argument is itself a function, **parenthesise** its type:

```gren
map : (a -> b) -> Array a -> Array b
applyTwice : (a -> a) -> a -> a
```

Without the parens, `a -> b -> Array a -> Array b` would mean a 3-argument function returning `Array b`.

## Type variables

Lowercase identifiers in type signatures are type variables: `a`, `b`, `msg`, `model`, `comparable`, `number`, `appendable`, `compappend`.

- **Unconstrained** (`a`, `b`, `msg`): can be any type, but every occurrence of the same variable must be the same type. In `map : (a -> b) -> Array a -> Array b`, all `a`'s match and all `b`'s match.
- **Constrained** (sometimes called "typeclasses" by analogy):
  - `number` allows `Int` or `Float` and admits `+`, `-`, `*`, `^`. (The `/` operator is **not** a `number` operation: it has type `Float -> Float -> Float` and forces both operands to `Float`. Use `//` for integer division or `toFloat`/`round` to mix types.)
  - `comparable` allows `Int`, `Float`, `Char`, `String`, and `Array`s of comparable values, and admits `<`, `>`, `<=`, `>=`. These are also the only types usable as `Dict` keys or `Set` members.
  - `appendable` allows `String` or `Array` and admits `++`.
  - `compappend` is the union of `comparable` and `appendable`.

You don't declare these — the compiler infers them from how the value is used.

## Records in types

Records are written `{ fieldName : FieldType, … }`. See `04-data-structures.md` for the full treatment. In a signature:

```gren
type alias User =
    { id : UserId
    , name : String
    , email : Email
    }

greet : User -> String
```

"Anonymous" records (without an alias) are also valid:

```gren
pointFrom : { x : Float, y : Float } -> Float
```

## Custom types in types

Custom types use `type` (not `type alias`). See `04-data-structures.md` § Custom types:

```gren
type ConnectionState
    = Disconnected
    | Connecting
    | Connected { sessionId : SessionId }
    | Failed { error : ConnectionError }
```

## `type alias` vs `type`

This trips up newcomers:

- `type alias Name = …` gives a **convenient name** to an existing type (often a record). It does not create a new type. Gren does **not** auto-generate a constructor for record aliases (this differs from Elm — see `09-differences-from-elm.md`).

  ```gren
  type alias Point = { x : Float, y : Float }
  -- `Point` and `{ x : Float, y : Float }` are interchangeable.
  ```

- `type Name = …` creates a brand new type with its own constructors. Use this for sum types and opaque types:

  ```gren
  type UserId = UserId Int
  ```

## The unit type

Gren's unit type is the empty record `{}`, **not** `()` (which Elm and Haskell use). The unit value is also `{}`.

```gren
noop : {} -> {}
noop _ = {}
```

Unit shows up rarely in everyday code; you will most often see it in ports or low-level library signatures where "no useful value" must be passed.

## Tuple types — don't

Gren removed tuples. Use a record with named fields instead:

```gren
-- not valid Gren
pair : ( Int, String )

-- idiomatic
type alias Pair = { quantity : Int, label : String }
```

See `09-differences-from-elm.md` for the rationale.

## Annotations you will commonly see

From a sample Node program (`src/Main.gren`):

```gren
update : Msg -> Model -> { model : Model, command : Cmd Msg }
subscriptions : Model -> Sub Msg
parseCommand : String -> Result InvalidCommand Command
navigate : Location -> Direction -> Maybe Location
```

The `{ model : Model, command : Cmd Msg }` is the anonymous record `update` returns in **every** Gren program, Node or browser. Gren has no tuples, so it cannot return the `(Model, Cmd Msg)` pair that Elm uses — both `Node.defineProgram` and `Browser.element` (and friends) use the same anonymous-record shape.

## When you can skip annotations

You can omit annotations entirely; the compiler infers everything. But:

- **Always annotate top-level functions** that appear in a module's `exposing` list. They are the public API.
- Annotating locals inside `let` is allowed but uncommon; inference usually suffices.

## Pitfalls

- Forgetting to parenthesise a function-typed argument: `map : a -> b -> Array a -> Array b` is **wrong** for the usual `map`. Write `(a -> b) -> Array a -> Array b`.
- Mixing `=` and `:` — annotation line uses `:`, definition line uses `=`.
- Writing `List Int` — Gren has no `List` type by default; the sequential data structure is `Array Int`. See `04-data-structures.md`.
- Expecting `()` for unit. Use `{}`.

## Cross-references

- Where to declare things: `01-values-and-functions.md`
- Data shapes you can annotate: `04-data-structures.md`
- Effects in signatures (`Cmd`, `Sub`, `Task`): `07-commands-and-subs.md`, `08-task-ports.md`
