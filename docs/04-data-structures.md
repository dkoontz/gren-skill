# 04 — Data Structures

How to model data in Gren: records, custom types, opaque types, and the standard collection modules (`Array`, `Dict`, `Set`). Also pointers on `String` and `Bytes`.

## Records

A record is a typed collection of named fields. Once a record's field names and types are fixed, they are fixed — Gren is structurally typed but records are not "open" by default.

### Construction

```gren
type alias User =
    { id : UserId
    , name : String
    , email : Email
    }

user : User
user =
    { id = UserId.fromInt 1
    , name = "Alice"
    , email = Email.fromString "alice@example.com"
    }
```

Field order in a literal does not need to match the alias, but every field must be present.

### No auto-constructor

Elm auto-generates a constructor function from a record type alias (`User id name email`). **Gren does not.** Always build records with the literal `{ … }` syntax. See `09-differences-from-elm.md`.

### Field access

Use a dot:

```gren
userName = user.name
```

You can also access fields as functions using the parenthesised form, which is useful in pipelines:

```gren
userName = user |> .name
names = Array.map .name users
```

### Record update

`{ record | field = newvalue }` returns a copy of the record with the named field(s) replaced:

```gren
updatedUser = { user | name = "Alicia" }

multiUpdate = { user | name = "Alicia", email = newEmail }
```

Gren extends record update beyond Elm: the left-hand side of `|` can be **any expression** that evaluates to a record, not just a variable name:

```gren
updated = { SomeModule.getRecord | value = "new" }
updated = { getConfig env | timeout = 30 }
```

See `09-differences-from-elm.md` for the contrast.

### Anonymous records in signatures

A function does not need a `type alias` to take a record — it can ask for the shape inline:

```gren
greet : { a | name : String } -> String
greet r = "Hello, " ++ r.name
```

This is the closest Gren comes to "structural row polymorphism": `{ a | name : String }` means "any record with at least a `name` field". This is extremely useful for keeping helper functions decoupled from your concrete record types.

## Custom types (sum types)

A custom type enumerates the possible variants of a value. Constructors start with a capital letter and are separated by `|`:

```gren
type ConnectionState
    = Disconnected
    | Connecting
    | Connected { sessionId : SessionId }
    | Failed { error : ConnectionError }
```

### Constructor arity: 0 or 1

Gren constructors take **0 or 1 argument**, never more. When you need to attach multiple pieces of data, wrap them in a record:

```gren
-- not valid Gren
type Person = Person String Int

-- valid
type Person = Person { name : String, age : Int }
```

The single argument may itself be any type, including an `Array`, a function, or another custom type.

### Using custom types — pattern matching

You consume custom types with `when … is` (see `03-decisions.md`):

```gren
describeState : ConnectionState -> String
describeState state =
    when state is
        Disconnected -> "Not connected"
        Connecting -> "Connecting..."
        Connected { sessionId } -> "Session " ++ SessionId.toString sessionId
        Failed { error } -> "Failed: " ++ ConnectionError.toString error
```

Real example from `src/Main.gren`:

```gren
type Command
    = Look
    | Go Direction
    | Take Item
    | Inventory
    | Help
    | Quit
```

Each variant either carries no data (`Look`, `Inventory`, `Help`, `Quit`) or one argument (a custom type). To create a `Command`, apply the constructor: `Go North`, `Take Keycard`.

### Constructors are functions

A constructor with one argument behaves like a function `a -> MyType`:

```gren
justOne : Maybe Int
justOne = Just 1

mapJust : Array Int -> Array (Maybe Int)
mapJust = Array.map Just
```

### Generic custom types

Custom types can take type parameters:

```gren
type Maybe a = Just a | Nothing
type Result e a = Err e | Ok a
```

## Opaque types

An opaque type is a custom type whose name is exported but whose constructors are **not**. The outside world can pass the type around and use the module's API; they cannot construct or pattern-match on it directly. This is how Gren enforces invariants:

```gren
module UserId exposing (UserId, fromInt, toString)

type UserId = UserId Int

fromInt : Int -> Maybe UserId
fromInt n =
    if n > 0 then Just (UserId n)
    else Nothing

toString : UserId -> String
toString (UserId n) = String.fromInt n
```

The `exposing` list names `UserId` (the type) but not `UserId(..)` (the constructor). Consumers must go through `fromInt`, which enforces the positivity invariant.

Use opaque types whenever a value should not be constructable with arbitrary contents — IDs, validated strings, state machines, etc.

## Making invalid states unrepresentable

A sum type enumerates what is *possible*; an argument position often needs only a *subset* of it. If `ConnectionState` has three variants but `startConnection` only makes sense for two of them, a plain `ConnectionState -> Task err {}` parameter still lets callers pass the third — the type system permits a state the function must then reject at runtime. Restructure so the argument type names only the valid options.

**Subtypes (the standard fix).** Factor the permitted variants into their own type; make the wide type include the narrow one:

```gren
type Attempting
    = Disconnected
    | Connecting

type ConnectionState
    = InProgress Attempting
    | Connected { sessionId : SessionId }
    | Failed { error : ConnectionError }

startConnection : Attempting -> Task ConnectionError ConnectionState
```

`InProgress` nests `Attempting`, so the wide type loses no expressiveness, and every case of `Attempting` is legal in `startConnection` — no impossible branch to write, no runtime guard. Widening is one constructor application (`InProgress state`); narrowing needs an explicit conversion function returning `Result`, which is where the restriction is enforced *once*, with a typed error, instead of at every call site.

**Phase parameters — phantom types.** Gren accepts a type variable that appears in the *type* but in no constructor argument; it carries no runtime value. That is a phantom type, and it is the tool for "same value, different permission":

```gren
type Machine phase
    = Machine Int          -- `phase` appears in no payload

type Open   = Open         -- phantom markers; nobody calls these constructors
type Closed = Closed

open      : {}             -> Machine Open
writeLine : Machine Open   -> Machine Open
close     : Machine Open   -> Machine Closed
flush     : Machine Closed -> Int
```

`writeLine (close handle)` is a **compile error**: `Machine Closed` does not unify with `Machine Open`. Enforcing that requires opacity — export `Machine`, never `Machine(..)`, or a caller writes `Machine 0` and claims any phase they like. Verified on Gren 0.6.5 in this repo: `tests/src/Test/PhantomMachine.gren` plus its suite in `Test/PhantomTypes.gren`; the same module with the phase parameter deleted lets the write-after-close code compile clean, so the phantom parameter — not a runtime check — is what rejects it.

The limit is informative: a phantom parameter is erased, so you cannot `when` on it, and it constrains *which phase*, not *which variant*. Use it when many functions accept different subsets of one type's states; use the subtype split above when one function needs a fixed variant subset.

**`Never` (mostly encountered, rarely deployed).** `Basics.Never` has no values, so a variant carrying one is unconstructable. `Basics.never : Never -> a` exists to eliminate it. As with Elm's `Html Never` idiom, `Never` is useful in *argument* position and a liability in return position — prefer subtypes or phantom phases.

Prefer opaque types (above) when the constraint is a value's *contents*; prefer this section when the constraint is *which variant or phase* a position may hold. They combine: an opaque newtype wrapping a narrowed sum type is a fine end state.

## Parse, don't verify

Unstructured data — `String`, `Int`, JSON, CLI args — becomes a domain value exactly once, at a boundary, via a parser that returns `Result domainValue error`. After the boundary, the value's type *is* the proof it was checked. "Verify" is the anti-pattern: keeping fields as raw `String`/`Int` and re-checking predicates (or worse, defaults) at each use site. Every use site then has to remember the check; a forgotten check compiles.

**Records full of `Maybe` are the smell.** A record with `Maybe` fields pushes "was this present/valid?" decisions onto every consumer:

```gren
-- weak: every consumer must re-check, and can forget
type alias UserDto =
    { id : Maybe Int
    , name : Maybe String
    , email : Maybe String
    }

-- strong: checks already happened; consumers can't forget
type alias User =
    { id : UserId
    , name : Name
    , email : Email
    }
```

Each strong field is an opaque type (`§ Opaque types`) whose only constructor path is a validating `fromString`/`fromInt`, so a `User` value that skipped validation is unconstructable — the type makes the invalid record unrepresentable. The parse that builds it fails hard and early, carrying a typed error (see `06-errors.md § Custom error types`):

```gren
parseUser : RawInput -> Result UserError User
parseUser raw =
    Ok (\id name email -> { id = id, name = name, email = email })
        |> Result.andMap (UserId.fromString raw.id |> Result.mapError BadId)
        |> Result.andMap (Name.fromString raw.name |> Result.mapError BadName)
        |> Result.andMap (Email.fromString raw.email |> Result.mapError BadEmail)
```

Record aliases have **no auto-constructor**, so the builder lambda above is required (`Ok User` alone is a compile error). With 2–3 fields, `Result.map2`/`map3` (which also need a lambda, not `User`) read better than `andMap`; see `06-errors.md § Combining`.

**Cross-field rules belong in the record's parse, not the field parsers.** When fields are valid only in combination — `from` before `until`, exactly one of `password`/`token` — field types alone cannot express it. Give the *record* an opaque type too, and enforce the combined invariant in its parse:

```gren
module BookingWindow exposing (BookingWindow, parse)

import Time exposing (Posix)

type BookingWindow = BookingWindow { from : Posix, until : Posix }

type BookingError
    = ReversedRange
    | Unparseable String

parse : { raw | from : String, until : String } -> Result BookingError BookingWindow
parse raw =
    -- Timestamp.fromIsoString is a project-level parser (core has no Date
    -- module; see 10-finding-source.md). The pattern is the point.
    when { from = Timestamp.fromIsoString raw.from, until = Timestamp.fromIsoString raw.until } is
        { from = Just from, until = Just until } ->
            if Time.posixToMillis from < Time.posixToMillis until then
                Ok (BookingWindow { from = from, until = until })
            else
                Err ReversedRange

        { from = Nothing } ->
            Err (Unparseable "from")

        { from = Just _, until = Nothing } ->
            Err (Unparseable "until")
```

A public `{ from : Posix, until : Posix }` record would let anyone build a reversed window; the opaque wrapper is what makes that unrepresentable. The three `when` branches are exhaustive over the two `Maybe` fields — no wildcard needed (Gren record patterns may omit fields, verified on 0.6.5).

**No `Date` module in core.** Time-related types in `gren-lang/core` are in `Time`: `Posix`, `Zone`, `Month`, `Weekday`, plus `posixToMillis`/`millisToPosix`. There is **no** `Date.fromString` like Elm has — Elm's `Date` module does not exist in Gren. ISO-8601 parsing is a project-level (or third-party-package) concern; the idiomatic home for it is an opaque `Timestamp` wrapper around `Time.Posix` whose only public entry point is a validating `fromIsoString : String -> Maybe Timestamp` — which is exactly this section's pattern applied to the field type itself. Do not silently substitute Elm's `Date` (see `09-differences-from-elm.md`).

**JSON boundaries.** `Json.Decode` composes the same way. `Json.Decode` keeps `map2`/`map3`/`map4`/`map5` (the "mapN was removed" rule does **not** apply here — check the pinned version per `10-finding-source.md`), and `andThen` is how a validating opaque constructor gets threaded in:

```gren
import Json.Decode as Decode

validated : (a -> Result String b) -> String -> Decode.Decoder a -> Decode.Decoder b
validated parse what decoder =
    decoder
        |> Decode.andThen
            (\raw ->
                when parse raw is
                    Ok value ->
                        Decode.succeed value

                    Err problem ->
                        Decode.fail (what ++ " is invalid: " ++ problem)
            )

userDecoder : Decode.Decoder User
userDecoder =
    Decode.map3
        (\id name email -> { id = id, name = name, email = email })
        (validated UserId.fromInt "id" (Decode.field "id" Decode.int))
        (validated Name.fromString "name" (Decode.field "name" Decode.string))
        (validated Email.fromString "email" (Decode.field "email" Decode.string))
```

A value that decodes has passed every check; there is no `Maybe` left to trip over downstream.

**Where parsing happens.** At ingress: decoding CLI args/HTTP bodies/file contents, `init`, and `update` branches that receive `String`/`Bytes`/`Value` from ports or streams. Everything downstream takes the strong type. The "fail hard" part is typed: `Result` + custom error (or an `Init.Task` rejection at startup), never `Debug.todo` or a silent default.

## Arrays

`Array a` is Gren's default sequential data structure — an immutable array, not a linked list. Gren removed `List` (which Elm kept).

### Literal syntax

```gren
numbers : Array Int
numbers = [ 1, 2, 3, 4, 5 ]

names : Array String
names = [ "alice", "bob" ]
```

Square brackets always denote an `Array`. There is no separate list literal.

### Common operations

`gren-lang/core` 7.4.2 exposing list (the project's pinned version per `gren.json`):

```gren
-- Transform
Array.map : (a -> b) -> Array a -> Array b
Array.indexedMap : (Int -> a -> b) -> Array a -> Array b
Array.foldl : (a -> b -> b) -> b -> Array a -> b
Array.foldr : (a -> b -> b) -> b -> Array a -> b
Array.keepIf : (a -> Bool) -> Array a -> Array a
Array.indexedKeepIf : (Int -> a -> Bool) -> Array a -> Array a
Array.mapAndKeepJust : (a -> Maybe b) -> Array a -> Array b
Array.mapAndFlatten : (a -> Array b) -> Array a -> Array b
Array.reverse : Array a -> Array a

-- Query
Array.isEmpty : Array a -> Bool
Array.length : Array a -> Int
Array.get : Int -> Array a -> Maybe a   -- negative indices count from the end
Array.findFirst : (a -> Bool) -> Array a -> Maybe { index : Int, value : a }
Array.findLast : (a -> Bool) -> Array a -> Maybe { index : Int, value : a }
Array.member : a -> Array a -> Bool
Array.any : (a -> Bool) -> Array a -> Bool
Array.all : (a -> Bool) -> Array a -> Bool
Array.minimum : Array comparable -> Maybe comparable
Array.maximum : Array comparable -> Maybe comparable

-- Modify
Array.set : Int -> a -> Array a -> Array a
Array.update : Int -> (a -> a) -> Array a -> Array a
Array.insert : Int -> a -> Array a -> Array a
Array.remove : Int -> Array a -> Array a
Array.pushFirst : a -> Array a -> Array a
Array.pushLast : a -> Array a -> Array a
Array.slice : Int -> Int -> Array a -> Array a

-- Deconstruct
Array.first : Array a -> Maybe a
Array.last : Array a -> Maybe a
Array.popFirst : Array a -> Maybe { first : a, rest : Array a }
Array.popLast : Array a -> Maybe { last : a, initial : Array a }
Array.partition : (a -> Bool) -> Array a -> { trues : Array a, falses : Array a }

-- Sort
Array.sort : Array comparable -> Array comparable
Array.sortBy : (a -> comparable) -> Array a -> Array a
Array.sortWith : (a -> a -> Order) -> Array a -> Array a
```

Things that **do not** exist (common gotchas):

- **No `dropIf`** — use `Array.keepIf (not << predicate)` to drop.
- **No `Array.count`** — it's `Array.length`.
- **No `Array.toList` / `fromList`** — `Array` is the default; there is no separate `List` to convert to/from.

### Renamed from Elm

Several functions were renamed for clarity in Gren:

| Elm (`elm-lang/core`)  | Gren              | Why                                |
| ------------ | ----------------- | ---------------------------------- |
| `filter`     | `keepIf`          | Describes what it does             |
| `filterMap`  | `mapAndKeepJust`  | Describes the two operations       |
| `concatMap`  | `mapAndFlatten`   | Describes the two operations       |
| `isEmpty`    | `isEmpty`         | Unchanged                          |
| `length`     | `length`          | Unchanged                          |

Note: some Elm packages use `flatMap` as an alias; in `elm-lang/core` the function is `concatMap`. Either way, the Gren equivalent is `mapAndFlatten`.

See `09-differences-from-elm.md`.

### Example from the project

In `src/Main.gren` around lines 172–180, the inventory is rendered with:

```gren
[ "You are carrying:" ]
    ++ (model.inventory
            |> Array.map (\item -> "  - " ++ itemName item)
            |> Array.foldr (\a acc -> Array.pushFirst a acc) []
       )
```

A coach might point out that the `foldr` + `Array.pushFirst` pattern rebuilds the array in reverse — since `Array.reverse` exists in `gren-lang/core` 7.4.2, the same effect is available via `Array.map f arr |> Array.reverse`. The current code is a useful teaching moment for discussing `foldl`/`foldr` direction, not a TODO.

## Dict

`Dict comparable v` is an immutable key/value map. Keys must be `comparable` — `Int`, `Float`, `Char`, `String`, or an `Array` of comparable values. (Gren has no tuples and no `List`, so the comparable kinds stop at those primitives plus arrays thereof.) Set, remove, and query operations all take O(log n) time.

```gren
import Dict exposing (Dict)

ages : Dict String Int
ages =
    Dict.empty
        |> Dict.set "alice" 30
        |> Dict.set "bob" 25
```

Note the function is **`Dict.set`**, not `Dict.insert`. (Gren's `Dict` and `Set` API both use `set` for the add-or-replace operation. Elm's `Dict.insert` has no exact namesake.) There is no built-in helper for building a `Dict` from a sequence of key/value pairs in 7.4.2; fold explicitly over an array of records:

```gren
fromKeyedRecords : Array { key : comparable, value : v } -> Dict comparable v
fromKeyedRecords records =
    Array.foldl (\r acc -> Dict.set r.key r.value acc) Dict.empty records
```

Common operations (`gren-lang/core` 7.4.2): `Dict.empty`, `Dict.singleton`, `Dict.set`, `Dict.update`, `Dict.updateWithDefault`, `Dict.remove`, `Dict.get`, `Dict.member`, `Dict.count`, `Dict.keys`, `Dict.values`, `Dict.map`, `Dict.foldl`, `Dict.foldr`. `Dict.get` returns `Maybe v`, which forces the caller to handle absence (see `06-errors.md`). Use `Dict.count` for the number of entries (there is no `Dict.size`).

## Set

`Set comparable` is an immutable set of `comparable` values. Internally a `Dict` with no values; same `comparable` constraint and the same O(log n) bounds. The add-or-replace function is **`Set.set`**, not `Set.insert` (matches `Dict.set`).

Common operations (`gren-lang/core` 7.4.2): `Set.empty`, `Set.singleton`, `Set.set`, `Set.toggle`, `Set.remove`, `Set.member`, `Set.count`, `Set.union`, `Set.intersect`, `Set.diff`, `Set.toArray`, `Set.fromArray`, `Set.map`, `Set.foldl`, `Set.foldr`, `Set.keepIf`, `Set.partition`. Use `Set.count` for the number of elements (there is no `Set.size`).

## String and Bytes

- `String` is UTF-8 text, immutable. Common: `String.trim`, `String.toUpper`, `String.toLower`, `String.toInt`, `String.fromInt`, `String.split`, `String.join`, `String.startsWith`, `String.dropFirst`, `String.length`.
- `Bytes` is a sequence of raw bytes, used by the I/O streams in `gren-lang/node`. Treat it as opaque; use the conversion helpers in the relevant library.

`String.toInt : String -> Maybe Int` — note the `Maybe`. Parsing can fail.

## Choosing a data structure

| Need                                   | Use                            |
| -------------------------------------- | ------------------------------ |
| A fixed set of cases                    | Custom type                    |
| A record with named fields              | Record (or `type alias`)       |
| A collection you iterate or transform   | `Array`                        |
| Lookup by key                           | `Dict`                         |
| Membership / uniqueness                 | `Set`                          |
| An optional value                       | `Maybe` (see `06-errors.md`)   |
| A computation that can fail             | `Result` (see `06-errors.md`)  |

There is **no List, no tuple, no linked-list**. If a user reaches for any of these, redirect to `Array` and records.

## Cross-references

- How to declare these in code: `01-values-and-functions.md`
- How to annotate them: `02-type-signatures.md`
- How to branch on them: `03-decisions.md`
- How to chain transformations on them: `05-pipeline-style.md`
