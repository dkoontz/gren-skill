# 07 — Commands and Subscriptions

Gren is a pure functional language, which means functions cannot perform side effects. Instead, side effects are **described as data** and handed to a runtime that executes them. The two description types are `Cmd msg` (do something) and `Sub msg` (listen for something).

This doc covers the runtime model at the language level. For HTTP, ports, and async I/O see `08-task-ports.md`.

## The runtime contract

A Gren program is a value handed to a runtime. Gren has two official runtimes:

- **Node** (`gren-lang/node`): the entry point is `Node.defineProgram`. Used for CLIs, servers, build tools, scripts — anything that runs in Node.js.
- **Browser** (`gren-lang/browser`): four entry points of increasing capability — `Browser.sandbox`, `Browser.element`, `Browser.document`, `Browser.application`. Used for anything that runs in a web page.

Both runtimes expect the same three things from you:

1. **`init`** — produce the initial model and an initial command.
2. **`update`** — given a message and the current model, produce a new model and a new command.
3. **`subscriptions`** — given the current model, declare which external events you want delivered as messages. (Not present in `Browser.sandbox`, which has no effects.)

This `init`/`update`/`subscriptions` loop is the architecture of every Gren program, Node or browser.

## `Cmd msg`

A `Cmd msg` describes a side effect to perform. It is **not executed** by your code — it is returned from `init`/`update`, and the runtime performs it. When the effect completes, the runtime produces a `msg` and feeds it back into `update`.

```gren
type Cmd msg   -- abstract; you never construct it directly
```

`Cmd` has two essential constructors you will use constantly:

```gren
Cmd.none : Cmd msg
Cmd.batch : Array (Cmd msg) -> Cmd msg
```

`Cmd.none` means "do nothing". `Cmd.batch` runs several commands in parallel.

Side-effecting operations are turned into `Cmd` via runners like `Task.execute` or `Task.attempt` (see `08-task-ports.md`), or library helpers like `Http.get`'s `expect*` family.

## `Sub msg`

A `Sub msg` describes an ongoing subscription — the runtime keeps it active and emits messages into your `update` as events arrive.

```gren
type Sub msg   -- abstract
```

`Sub.none` and `Sub.batch` mirror their `Cmd` counterparts.

Subscriptions are typically things like:

- Browser: keyboard events, window resize, mouse/pointer events, animation frames, websocket messages.
- Node: OS signals (e.g. `Node.onSignalInterrupt` for SIGINT — see `src/Main.gren` around line 205), incoming socket data, file watchers.

Your `subscriptions` function returns the current set, which can change as the model changes.

## The `init` / `update` / `subscriptions` signatures

The exact shapes differ between Node and browser, but both return the model and command in an **anonymous record** `{ model, command }` — never a tuple, because Gren has no tuples.

### Node

```gren
main : Node.Program Model Msg
main =
    Node.defineProgram
        { init = init
        , update = update
        , subscriptions = subscriptions
        }

init : Node.Environment -> Init.Task { model : Model, command : Cmd Msg }
update : Msg -> Model -> { model : Model, command : Cmd Msg }
subscriptions : Model -> Sub Msg
```

Node's `init` is wrapped in `Init.Task` so it can request sub-system permissions (terminal, file system, network, etc.) before the program starts. The `Node.Environment` argument carries handles like `stdin`/`stdout`. See the project's `src/Main.gren` `main` and `init` (around lines 79–108) for a concrete example.

### Browser

The four program constructors trade off capability for complexity:

| Constructor             | Capabilities                                                                                                         |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `Browser.sandbox`       | `init` + `update` + `view`. No `Cmd`, no `Sub`, no `flags`. For learning and pure UI experiments.                    |
| `Browser.element`       | Adds `Cmd`, `Sub`, and `flags`. Embeds a Gren-managed element inside a larger JS page.                               |
| `Browser.document`      | Same as `element` but `view` controls `<head>` and `<title>` (returns `Browser.Document msg`).                       |
| `Browser.application`   | Same as `document` plus single-page-app routing via `onUrlRequest`/`onUrlChange`.                                    |

Verified signatures for `Browser.element` (latest `gren-lang/browser`):

```gren
main : Program flags Model Msg
main =
    Browser.element
        { init = init
        , view = view
        , update = update
        , subscriptions = subscriptions
        }

init : flags -> { model : Model, command : Cmd Msg }
view : Model -> Html Msg
update : Msg -> Model -> { model : Model, command : Cmd Msg }
subscriptions : Model -> Sub Msg
```

`Browser.application` adds `init : flags -> Url -> Key -> { model, command }` (the `Url` is the starting URL; `Key` is for navigation) and `onUrlRequest : UrlRequest -> msg`, `onUrlChange : Url -> msg`.

### Things to note across both runtimes

- `update` always returns `{ model : Model, command : Cmd Msg }`, never a tuple.
- `Msg` is just a custom type. Each variant corresponds to one kind of event your program can receive.
- Browser programs include a `view` function (Node programs don't — Node has no DOM).

## `Msg` design

`Msg` is the catalogue of events the program can react to. Constructors typically correspond to:

- **External events**: `GotResponse (Result Http.Error String)`, `LineRead (Result Stream.Error String)`, `KeyPressed KeyCode`.
- **Subscription events**: `Interrupted`, `Resized Int Int`.
- **Internal events**: anything the program needs to schedule as a follow-up to itself.

Real example from `src/Main.gren` around lines 69–73:

```gren
type Msg
    = LineRead (Result Stream.Error String)
    | Interrupted
    | SeedGenerated Int
```

`LineRead` carries the outcome of an attempted line read. `Interrupted` is a no-data message from the OS signal subscription. `SeedGenerated` carries a random number generated as a side effect on startup.

## How a single effect flows

Concretely, in the sample Node program (paraphrased from `src/Main.gren`):

1. `init` issues a `Cmd.batch` that includes `Stream.readBytesAsString env.stdin |> Task.attempt LineRead` — i.e. "read a line from stdin; when it arrives, hand me a `LineRead (Result …)` message."
2. The runtime performs the read asynchronously.
3. When input arrives, the runtime constructs `LineRead (Ok "go north")` (or `LineRead (Err …)` on EOF) and feeds it to `update`.
4. `update` pattern-matches on the message, updates the model, and returns the next command — which often is "read another line" (see `respond` around lines 189–200).

In a browser program the shape is identical, only the effect source differs: `init` might issue `Http.get { url = ..., expect = Http.expectString GotText }`, the runtime fires the request, and on completion `GotText (Result Http.Error String)` is fed back into `update`.

This loop is the spine of every Gren program, Node or browser. There is no event emitter, no callback registry — just `update` returning the next `Cmd`.

## Batching and ordering

`Cmd.batch [ a, b, c ]` issues all three commands. They run concurrently; their resulting messages arrive in whatever order the runtime resolves them. Your `update` must be prepared for any order.

If you need **ordering** — "do A, then once A is done, do B" — you need `Task` chains (see `08-task-ports.md`). `Cmd.batch` does not sequence.

## `Cmd.none` vs no command

A function whose return type is `{ model : Model, command : Cmd Msg }` must always provide a `command`. When you have nothing to do, return `Cmd.none`. There is no `Maybe Cmd` — the field is always present.

## Where effects actually live

| Effect family              | Module(s) to look in                                 | Runtime            |
| -------------------------- | ---------------------------------------------------- | ------------------ |
| HTTP requests              | `Http`                                               | browser            |
| Random numbers             | `Random`                                             | core (both)        |
| Time / clock               | `Time`                                               | core (both)        |
| Browser DOM / events       | `Html`, `Html.Attributes`, `Html.Events`, `Browser`  | browser            |
| File / stream I/O          | `Stream`, `Stream.Log`, other `Node.*` sub-systems   | node               |
| Process control (exit/env) | `Node`                                               | node               |
| File system access         | `FileSystem` and related sub-systems                  | node               |
| JavaScript interop         | Ports — see `08-task-ports.md`                       | both               |

For exact function surfaces inside any module, browse the per-module docs at `https://packages.gren-lang.org/package/<author>/<package>/version/<version>/module/<Module>` (see `10-finding-source.md`).

## Cross-references

- Tasks, ports, and async: `08-task-ports.md`
- Pattern matching on `Msg`: `03-decisions.md`
- Modelling `Msg` and model as custom/record types: `04-data-structures.md`
- What the project actually does: `src/Main.gren` (esp. `main`, `init`, `update`, `subscriptions`)
