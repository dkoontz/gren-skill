---
name: gren
description: Gren language reference for reading, writing, and explaining Gren code. Covers syntax, types, data structures (Array/Dict/Set), pipeline style, Maybe/Result error handling, the Cmd/Sub/Task effect model, and ports. Load this when working with .gren files, debugging Gren, or explaining how Gren code works.
---

# Gren Language

You now have reference knowledge of the **Gren** programming language. Gren is a pure functional language that compiles to JavaScript. Present and write syntax as **Gren syntax** — Gren has evolved into its own language; do not silently substitute constructs from Elm or other ML-family languages (many do not compile in Gren; see `./docs/09-differences-from-elm.md`).

## Characteristics

- **Pure functional** — no side effects inside functions; all effects are explicit
- **Statically typed** with full type inference — annotations are optional but recommended
- **No runtime exceptions** — no null, no `throw`/`try`/`catch`; absence is `Maybe`, failure is `Result`
- **Compiles to JavaScript** — runs on Node.js (`gren-lang/node`) or in the browser (`gren-lang/browser`)
- **Immutable** — you build new values instead of mutating

## How to use this knowledge

This skill gives you the essentials. For deeper detail on any topic, **lazily read the focused reference docs** bundled with this skill under `./docs/`. Load a doc only when a question touches its topic — do not dump a doc's contents at the user; internalise and answer in your own words.

| Topic                                                | Doc (under `./docs/`)                                             |
| ---------------------------------------------------- | ----------------------------------------------------------------- |
| Naming, `let`, function definitions, comments        | [`01-values-and-functions.md`](./docs/01-values-and-functions.md) |
| Type annotations, aliases, type variables, unit `{}` | [`02-type-signatures.md`](./docs/02-type-signatures.md)           |
| `if`, `when ... is`, all pattern-matching forms      | [`03-decisions.md`](./docs/03-decisions.md)                       |
| Records, custom types, Array, Dict, Set              | [`04-data-structures.md`](./docs/04-data-structures.md)           |
| `\|>`, data-last functions, chaining                 | [`05-pipeline-style.md`](./docs/05-pipeline-style.md)             |
| Maybe, Result, custom error types, `andMap`          | [`06-errors.md`](./docs/06-errors.md)                             |
| Cmd, Sub, the runtime, Node/browser programs         | [`07-commands-and-subs.md`](./docs/07-commands-and-subs.md)       |
| Task, Task ports, JS interop, concurrency            | [`08-task-ports.md`](./docs/08-task-ports.md)                     |
| Coming from Elm or another ML-family language        | [`09-differences-from-elm.md`](./docs/09-differences-from-elm.md) |
| Where to find Gren source, packages, or API docs     | [`10-finding-source.md`](./docs/10-finding-source.md)             |

When unsure which doc covers a question, start with [`./docs/README.md`](./docs/README.md).

---

## Values and functions

Top-level declarations are `name = expression`. No `let`/`const`/`var`/`def` keyword. Indentation is significant; a new declaration begins at column 0.

```gren
name = "Chorus"
count = 42
enabled = True

add x y = x + y
result = add 1 2          -- 3 (juxtaposition; no parens around args)
increment = add 1         -- partial application (curried)
double = \n -> n * 2      -- lambda
```

`let ... in ...` introduces local bindings (it is an expression — returns the `in` value):

```gren
processData data =
    let
        cleaned = String.trim data
        parsed = parseInput cleaned
    in
    validate parsed
```

**Naming:** values/functions are `camelCase`; types and constructors are `PascalCase`; modules are `PascalCase` and match the file name. Hyphens are **not** allowed in identifiers.

**Comments:** `--` single line; `{- ... -}` multi-line (nestable).

**Operators:** `+ - * / // ^ == /= < > <= >= && || ++ |> <| >> <<`. Note `/` is always `Float` division; `//` is integer division. Use `/=` for inequality (`!=` is a syntax error). `++` appends `String` or `Array`.

## Type signatures

Annotations are `name : Type`, on the line above the definition.

```gren
add : Int -> Int -> Int
add x y = x + y

map : (a -> b) -> Array a -> Array b
```

Arrows are right-associative (`Int -> Int -> Int` means `Int -> (Int -> Int)`). Parenthesise function-typed arguments: `(a -> b) -> Array a -> Array b`.

**Basic types:** `Int`, `Float`, `Bool` (`True`/`False`), `String`, `Char`, `Bytes`.

**Type variables:** lowercase (`a`, `b`, `msg`). Constrained: `number` (Int/Float, admits arithmetic), `comparable` (Int/Float/Char/String/Array of those, admits comparison and is required for Dict keys/Set members), `appendable` (String/Array, admits `++`).

**`type alias`** names an existing type (often a record) — it does **not** create a constructor. **`type`** creates a new type with constructors.

**Unit is `{}`** — the empty record is both the unit type and the unit value. (Other ML-family languages such as Haskell and Elm use `()`; Gren does not.)

## Decisions: `if` and `when`

`if` requires `else` (every expression returns a value); the condition must be `Bool` (no truthy/falsy).

```gren
sign n =
    if n < 0 then "negative"
    else if n == 0 then "zero"
    else "positive"
```

`when ... is` is Gren's pattern matching:

```gren
when maybeValue is
    Just x -> x
    Nothing -> default
```

`when` must be **exhaustive** — cover every constructor, or add a `_` wildcard for open types (String, Int, etc.). Supports literal, variable-binding, custom-type, record, nested, and wildcard patterns. **No guards** (`| pred ->` is invalid) — use `if` inside a branch, or match on a `Bool`.

## Data structures

### Records

```gren
type alias User = { id : UserId, name : String, email : Email }

user = { id = UserId.fromInt 1, name = "Alice", email = Email.fromString "a@b.com" }
userName = user.name
updated = { user | name = "Alicia" }            -- update returns a copy
updated = { getConfig env | timeout = 30 }      -- LHS can be ANY expression (not just a var)
```

No auto-constructor — always use the `{ ... }` literal. Row-polymorphic signatures: `{ a | name : String }` means "any record with at least `name`".

### Custom types (sum types)

```gren
type ConnectionState
    = Disconnected
    | Connecting
    | Connected { sessionId : SessionId }
    | Failed { error : ConnectionError }
```

**Constructors take 0 or 1 argument — never more.** Wrap multiple values in a record. Constructors are functions (`Just : a -> Maybe a`). Generic: `type Maybe a = Just a | Nothing`.

**Opaque types:** export the type name but not its constructors (`exposing (UserId)` not `UserId(..)`) to enforce invariants via a smart-constructor API.

**Making invalid states unrepresentable:** when an argument only permits a subset of a sum type's variants, restructure — a nested subtype (`InProgress Attempting`), a phantom phase parameter (`Machine phase`, opaque-wrapped; `writeLine (close h)` then fails to compile), or both. → `./docs/04-data-structures.md § Making invalid states unrepresentable`

**Parse, don't verify:** convert `String`/`Int`/JSON into validating opaque types once, at the boundary, returning `Result` — never a record of `Maybe` fields that every consumer must re-check. Cross-field invariants get an opaque record type. → `./docs/04-data-structures.md § Parse, don't verify`

### Array (not List)

`Array a` is the default sequential structure — an immutable array. **There is no `List`.** Square brackets always produce an `Array`:

```gren
numbers : Array Int
numbers = [ 1, 2, 3 ]

doubled = numbers |> Array.map (\n -> n * 2)
evens = numbers |> Array.keepIf (\n -> modBy 2 n == 0)
sum = numbers |> Array.foldl (+) 0
```

Key functions: `map`, `keepIf`, `mapAndKeepJust`, `mapAndFlatten`, `foldl`, `foldr`, `get` (returns `Maybe`), `set`, `pushLast`, `slice`, `isEmpty`, `length`, `reverse`, `findFirst`. There is **no `dropIf`** (use `keepIf (not << pred)`), **no `count`** (it's `length`).

### Dict and Set

`Dict comparable v` — keys must be `comparable`. Use **`Dict.set`** (not `insert`), `Dict.get` (returns `Maybe`), `Dict.count` (not `size`). `Set comparable` — use **`Set.set`** (not `insert`), `Set.count`.

## Pipeline style

Gren's central idiom: **data-last** functions chained with `|>`.

```gren
result =
    "hello"
        |> String.append " world"
        |> String.trim
        |> String.toUpper
```

Every well-designed Gren function puts configuration first and the principal data last, so `|>` reads top-to-bottom. A pipeline step can be a function name, a partially-applied function, a field accessor (`.name`), or a lambda. `<|` (reverse pipe) exists but is rare. `>>` and `<<` are function composition.

## Errors: Maybe and Result

No null, no exceptions. Absence is `Maybe`; failure-with-a-reason is `Result`.

```gren
type Maybe a = Nothing | Just a
type Result error value = Err error | Ok value
```

Consume by pattern matching or helpers: `Maybe.map`, `Maybe.andThen`, `Maybe.withDefault`, `Result.map`, `Result.andThen`, `Result.mapError`, `Result.fromMaybe`.

Gren removed `map4`/`map5`/`mapN`. Use `map` + `andMap`:

```gren
Maybe.map buildRecord
    |> Maybe.andMap maybeA
    |> Maybe.andMap maybeB
    |> Maybe.andMap maybeC
```

Rule of thumb: if you'd attach a string explaining _why_ a `Maybe` is `Nothing`, use `Result` with a custom error type instead. Side-effecting failures (HTTP, I/O) happen in `Task`/`Cmd` and arrive in `update` as `Msg` carrying a `Result`.

## Commands, subscriptions, and the runtime

Functions cannot perform side effects. Effects are **described as data** (`Cmd msg` / `Sub msg`) and handed to a runtime that executes them. Two runtimes:

- **Node** — `Node.defineProgram`. `init` is wrapped in `Init.Task` to request sub-system permissions.
- **Browser** — `Browser.sandbox` (no effects), `.element`, `.document`, `.application`.

Both expect `init`, `update`, `subscriptions` (sandbox omits subscriptions). The program returns model + command in an **anonymous record** — never a tuple (Gren has no tuples):

```gren
update : Msg -> Model -> { model : Model, command : Cmd Msg }
```

`Cmd.none` / `Cmd.batch` (runs in parallel, unordered), `Sub.none` / `Sub.batch`. `Msg` is a custom type cataloguing events. Effects loop: `update` returns the next `Cmd`; the runtime performs it and feeds a `Msg` back into `update`.

## Tasks and ports

`Task err ok` is a recipe for an asynchronous, fallible computation (error type first, matching `Result`). It is inert until turned into a `Cmd`:

- `Task.attempt : (Result x a -> msg) -> Task x a -> Cmd msg` — fallible work (callback **first**; idiomatic: `task |> Task.attempt Tag`)
- `Task.perform : (a -> msg) -> Task Never a -> Cmd msg` — infallible, need the value
- `Task.execute : Task Never a -> Cmd msg` — infallible, fire-and-forget

`Task.perform`/`execute` require `Task Never a` (cannot fail by type) — this is enforced, not a "drop errors" shortcut.

**Concurrency** `Task.map2`/`map3`/`andMap` run **concurrently** (and cancel in-flight tasks on failure). Use `Task.sequence` for serial, `Task.concurrent` for a parallel array.

**Ports** are the only way to call JavaScript. Three styles: outgoing `Cmd` ports, incoming `Sub` ports (classic, two declarations per effect), and **task ports** (`port saveToDB : String -> Task Json.Decode.Value Int` — request/response in one declaration, preferred for request/response interop).

## Syntax gotchas

The Gren forms that most often surprise readers coming from other ML-family languages (especially Elm):

- Pattern matching is `when … is` — `case … of` does not exist in Gren.
- **No tuples.** Use records with named fields everywhere, including `update`'s return: `{ model = …, command = … }`.
- The sequential structure is `Array a` (literal `[ 1, 2, 3 ]`); **there is no `List`.**
- Custom type constructors take **0 or 1 argument** — wrap multiple values in a record.
- Record aliases define **no constructor function**; build records with the `{ … }` literal.
- Filtering is `keepIf`; combine-and-filter is `mapAndKeepJust`; flatten-mapping is `mapAndFlatten`.
- `Dict.set` / `Set.set` for add-or-replace; `Dict.count` / `Set.count` for size.
- Unit is `{}`, not `()`.
- No `map4`/`map5` in core modules — use `map` + `andMap`.
- `Task.map2`/`map3`/`andMap` run their tasks **concurrently**.

The `Model`/`Msg`/`update`/`view` architecture, immutability, `Maybe`/`Result`, currying, `|>`, the `module … exposing`/`import` system, and annotation syntax are standard Gren. For the full Elm-to-Gren mapping, see `./docs/09-differences-from-elm.md`.

## Reading dependency source

**Read dependency source from `gren_packages/` — never from the global cache and never from the web for source.** The local `gren_packages/` directory is the only source guaranteed to match the versions pinned in `gren.json`.

Always check `gren.json` first. There are two kinds of dependencies:

**Registry packages (`.pkg` files)** — stored as a single JSON file named `<author>__<package>__<version>.pkg` (e.g. `gren_lang_core__7_4_2.pkg`). It is **not** an extracted source tree. The JSON has `outline` (metadata + `exposed-modules`) and `sources` (module name → full source string, including `Gren.Kernel.*`). Use `jq`:

```bash
jq '.sources | keys' gren_packages/gren_lang_core__7_4_2.pkg          # list modules
jq -r '.sources.Array' gren_packages/gren_lang_core__7_4_2.pkg         # extract a module
```

Prefer the uncompressed `.pkg` when it exists (bare `jq` works). Fall back to `gunzip -c file.pkg.gz | jq` only when there's no plain `.pkg`.

**Local packages (`local:./<dir>`)** — real source on disk. Read with normal file tools (`ls gren-lang-node/src`, open `.gren` files directly).

To locate a function across all deps, grep both locations: `rg "keepIf" gren_packages/ gren-lang-node/src`. (Inside a `.pkg`, source is one long JSON string — grep locates which package/module, then use `jq -r` to read it.)

## Verifying things you don't know

- Read `gren.json` for dependencies, `source-directories`, `platform` (`"node"`/`"browser"`), and `gren-version`.
- Map a module to a package: `Array`/`Result`/`Task`/`String`/`Dict`/`Set`/`Json.*`/`Math`/`Basics`/`Random`/`Time`/`Bytes` → `gren-lang/core`; `Node`/`Stream`/`FileSystem`/`Init`/`Terminal` → `gren-lang/node`; `Browser`/`Html`/`Http` → `gren-lang/browser`.
- Browse package docs at `https://packages.gren-lang.org/package/<author>/<package>/version/<version>/module/<Module>`.
- Use `rg` to map a project: `rg "^.* : " src` (all annotations), `rg "^(type|type alias) " src` (all types), `rg "\bparseCommand\b" src` (usages).

## Authoritative references

- Language guide: <https://gren-lang.org/book/>
- Core library: <https://github.com/gren-lang/core>
- Node runtime: <https://github.com/gren-lang/node>
- Browser runtime: <https://github.com/gren-lang/browser>
- Package catalogue: <https://packages.gren-lang.org/package>
