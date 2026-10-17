# Gren Coach — Reference Index

This directory holds focused reference documents the **Gren Coach** agent loads on demand. Each file covers one area of the language. The agent should read only what a given question requires; this file is the map.

> These are **general Gren language docs**. They cover both runtimes Gren officially targets — Node (`gren-lang/node`) and the browser (`gren-lang/browser`).
>
> **Verification baseline.** Language-level claims are checked against the compiler docs at <https://gren-lang.org/book/> and the FAQ at <https://gren-lang.org/book/appendix/faq/>. Function signatures for `gren-lang/core` are pinned to **7.4.2**, `gren-lang/node` to **6.1.3**, and `gren-lang/browser` to the **latest** published version (the docs note this where it matters). If the project you are coaching depends on different versions (check `gren.json`), re-verify API surfaces against the package docs before relying on a function list — see `10-finding-source.md`.
>
> **Examples.** Code samples marked "from a project" are illustrative; they come from a Node-based sample (`src/Main.gren` in this repo) and the line numbers are accurate as of the version in this repo. They are not normative.

## How the coach uses these docs

1. Match the user's question to a topic below.
2. `read` the relevant doc before answering.
3. Internalise, then answer in your own words with focused examples. Do not echo the doc.

## Document map

| File                              | Covers                                                                                          | When to load                                                                                |
| --------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| `01-values-and-functions.md`      | Top-level declarations, constants, function definitions, application, lambdas, `let`, comments | "How do I declare…", "what's a lambda", naming, scoping                                     |
| `02-type-signatures.md`           | Type annotations, basic types, function arrows, type variables, aliases vs custom types        | "What does this signature mean", "how do I write the type for…"                             |
| `03-decisions.md`                 | `if`/`then`/`else`, `when … is`, all pattern matching forms                                     | "How does `when` work", "can I match on a record", branching logic                          |
| `04-data-structures.md`           | Records, custom types (sum types), opaque types, phantom phases, parse-don't-verify, Array, Dict, Set, String/Bytes pointers       | "How do I model this data", "what data structure for…", record update, restricting valid variants/states, validating input at boundaries                       |
| `05-pipeline-style.md`            | The `\|>` operator, data-last argument order, why pipelines are idiomatic                        | "How do I chain these operations", "why is the data last"                                   |
| `06-errors.md`                    | `Maybe`, `Result`, custom error types, `andMap`, when to use which                              | "How do I handle missing values", "how do I model failure", parsing, validation             |
| `07-commands-and-subs.md`         | The Gren runtime, `Cmd`/`Sub`, `init`/`update`/`subscriptions`, Node and browser programs | "How do effects work", "where do side effects come from", message flow                      |
| `08-task-ports.md`                | `Task`, `Task.execute`/`attempt`, concurrent vs sequential, ports (Cmd/Sub/Task) for JS interop | "How do I do an HTTP request", "how do I call JS", "do tasks run in parallel"               |
| `09-differences-from-elm.md`      | Migration notes: syntax renames, removed features, idiomatic shifts                             | User mentions Elm, or you suspect they are copying Elm code                                 |
| `10-finding-source.md`            | Where Gren source lives locally and online, how to look up package APIs                         | "What does `Array.foo` do", "where is the source for this function", "which package provides…" |

## Conventions used across these docs

- All code blocks marked `gren` are Gren source. Where contrast is needed, `elm` and `javascript` blocks show what **not** to write.
- "Gren" always means the Gren language, never Elm.
- Examples marked "from a project" come from the Node-based sample in this repo (`src/Main.gren`); line numbers track that file. See the callout above.
- Cross-references between docs use the form `see 04-data-structures.md § Custom types`.

## Coverage

These docs cover the **Gren language** itself plus the standard libraries most programs touch: `Array`, `Maybe`, `Result`, `Task`, `Dict`, `Set`, `String`, `Bytes`, `Basics`, `Math` (all from `gren-lang/core`); the `Cmd`/`Sub`/`Task` effect model; and the program constructors for both runtimes (`Node.defineProgram` for Node, `Browser.sandbox`/`element`/`document`/`application` for the browser). HTTP, ports, and `Html` are covered at the level needed to read and write typical code; deeper module surfaces (`Html.Attributes`, `Json.Decode`, `Stream.Log`, etc.) are looked up on demand via `10-finding-source.md`.
