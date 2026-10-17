# 01 — Values and Functions

How you declare named things in Gren: constants, functions, local bindings, and the conventions around naming and comments.

## Top-level declarations

Every named value at the top level of a module has the shape `name = expression`. There is no `let`, `const`, `var`, or `def` keyword. Indentation is significant: a new top-level declaration begins at column 0.

```gren
name = "Chorus"
count = 42
enabled = True
```

A declaration may optionally be preceded by a type annotation (see `02-type-signatures.md`). When present, the annotation sits on its own line directly above the definition:

```gren
name : String
name = "Chorus"
```

## Naming conventions

- **Identifiers are `camelCase`** for values and functions: `parseCommand`, `randomNumberSeed`.
- **Types are `PascalCase`**: `ConnectionState`, `UserId`, `Msg`.
- **Modules are `PascalCase`** and match their file name: `module Main` lives in `src/Main.gren`.
- Constructors of custom types are `PascalCase`: `Just`, `Nothing`, `GotResponse`.
- Hyphens are **not** allowed in identifiers. Use `myValue`, never `my-value`.

## Constants

A "constant" is just a top-level value. Gren does not distinguish compile-time constants from other bindings — everything is immutable.

```gren
piApprox = 3.14159
greeting = "Hello, explorer."
```

## Functions

### Definition

A function is a value that takes arguments. Arguments are listed after the name, with no parentheses and no commas:

```gren
add x y = x + y
```

`add` has type `Int -> Int -> Int`. There is no `function` keyword and no arrow in the definition — the arrow form appears only in type signatures.

### Application

Function application is **juxtaposition** — write the function, then each argument separated by spaces. No parentheses around arguments, no commas:

```gren
result = add 1 2          -- 3
sum = add (add 1 2) 3     -- 6, parens group the inner call
```

Parentheses are only needed to disambiguate nested calls. They do **not** invoke a function the way they do in C or JavaScript.

### Partial application

Because functions are curried, you can pass fewer arguments than the function expects and get back a function awaiting the rest:

```gren
increment = add 1
five = increment 4
```

This is heavily used in pipelines (see `05-pipeline-style.md`).

### Anonymous functions (lambdas)

Use `\args -> body`:

```gren
increment = \x -> x + 1
combine = \x y -> x + y * 2
```

This is equialent to the regular function

```gren
increment x = x + 1
combine x y = x + y * 2
```

Lambdas are common as arguments to `Array.map`, `Array.keepIf`, etc.:

```gren
doubled = Array.map (\n -> n * 2) numbers
```

## Multi-line functions and `let`

For anything non-trivial, write the body across lines using indentation. Local bindings go inside a `let … in …` block:

```gren
processData data =
    let
        cleaned =
            String.trim data

        parsed =
            parseInput cleaned
    in
    validate parsed
```

Rules of `let`:

- Every binding inside the `let` block is indented relative to it.
- The `in` keyword starts the expression that uses those bindings.
- Bindings can reference each other; order is flexible but readability matters.
- `let` is an expression — it returns the value of the `in` expression.

Example from the project (see `src/Main.gren` around lines 117–123):

```gren
let
    parsed =
        line
            |> String.trim
            |> String.toLower
            |> parseCommand
in
...
```

## Operators

The full operator set lives in the `Basics` module of `gren-lang/core` and is imported by default. Source of truth: <https://github.com/gren-lang/core/blob/main/src/Basics.gren>.

| Operator          | Purpose                                                                        |
| ----------------- | ------------------------------------------------------------------------------ |
| `+` `-`           | Numeric add / subtract (works on `Int` or `Float`, not strings)                |
| `*`               | Numeric multiply                                                               |
| `/`               | **Floating-point** division (result is always `Float`)                         |
| `//`              | **Integer** division (truncates toward zero; result is `Int`)                  |
| `^`               | Exponentiation (`2 ^ 10 == 1024`)                                              |
| `==` `/=`         | Structural equality / inequality (do not use on functions or JSON)             |
| `<` `>` `<=` `>=` | Comparison (only on `comparable`: numbers, `Char`, `String`, `Array` of those) |
| `&&` `\|\|`       | Boolean AND / OR (short-circuit when used as infix)                            |
| `++`              | Append `String` or `Array`                                                     |
| `\|>`             | Pipe left-to-right: `x \|> f == f x` (see `05-pipeline-style.md`)              |
| `<\|`             | Pipe right-to-left: `f <\| x == f x` (rare; usually avoid)                     |
| `>>` `<<`         | Function composition (`>>` left-to-right, `<<` right-to-left)                  |

`xor : Bool -> Bool -> Bool` exists as a **function**, not an infix operator.

Precedence (lowest to highest): `(<|)` `(|>)` → `(||)` → `(&&)` → comparison and equality → `(++)` → `(+)` `(-)` → `(*)` `(/)` `(//)` → `(^)` → `(<<)` `(>>)`. This matches the operator table in `gren-lang/core`'s `Basics` module. When in doubt, parenthesise.

Use `/=` for "not equals" — `!=` is a syntax error.

Other helpers from `Basics` (not operators but imported by default): `negate`, `toFloat`, `isNaN`, `isInfinite`, `min`, `max`, `clamp`, `compare`, `Order` (LT/EQ/GT), `not`, `identity`, `Never`, `never`. The arithmetic helpers `abs`, `round`, `floor`, `ceiling`, `truncate`, plus `sqrt`, `sin`, `cos`, `logBase`, etc. live in the **`Math`** module of `gren-lang/core` (not `Basics`), so you must `import Math` to use them.

## Comments

Two forms:

```gren
-- a single-line comment

{- a multi-line
   comment
   can span many lines -}
```

`{- … -}` comments can also be nested, which is useful for commenting out blocks that themselves contain comments:

```gren
{- outer
   {- inner -}
   outer continued -}
```

There is no doc-comment syntax distinct from `--`. Documentation for published modules is generated from type annotations and the `exposing` list.

## What's deliberately not here

- **No mutation.** You cannot reassign a name. To "update" a value, build a new one (records make this cheap — see `04-data-structures.md`).
- **No `var`/`let`/`const` distinction.** All top-level and `let` bindings are immutable.
- **No statement/expression split.** Everything is an expression. `if`, `when`, `let` all return values (see `03-decisions.md`).
- **No null.** Optional values use `Maybe` (see `06-errors.md`).

## Cross-references

- Type annotations: `02-type-signatures.md`
- Branching (`if`, `when`): `03-decisions.md`
- Why arguments are ordered data-last: `05-pipeline-style.md`
