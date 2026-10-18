# 08 — Tasks and Ports

`Task` is Gren's type for asynchronous, fallible computation. Ports are its mechanism for talking to JavaScript. This doc covers both, plus the **concurrency semantics**, which are Gren's own and easy to guess wrong.

## The `Task` type

`Task err ok` describes a computation that will eventually:

- succeed with an `ok` value, or
- fail with a typed `err` value.

It looks similar to `Result err ok`, but a `Task` has not run yet — it is a *recipe* for an effect. The runtime performs it.

```gren
type Task err ok   -- abstract
```

Note the argument order: error first, success second. This matches `Result`.

A `Task` stays inert until you convert it into a `Cmd` via one of the runners (`gren-lang/core` 7.4.2):

| Runner            | Type                                            | Produces                                    |
| ----------------- | ----------------------------------------------- | ------------------------------------------- |
| `Task.perform`    | `(a -> msg) -> Task Never a -> Cmd msg`         | Run an infallible task; wrap its success value in a `msg` |
| `Task.execute`    | `Task Never a -> Cmd msg`                       | Run an infallible task whose result you don't need (e.g. logging). The `Task Never a` argument means the task **cannot fail by type** — `execute` does not "drop" errors, it requires them to be impossible |
| `Task.attempt`    | `(Result x a -> msg) -> Task x a -> Cmd msg`    | Run a fallible task; report `Ok`/`Err` as a `Msg`. Note the argument order: the callback is **first**, the task is second — so the idiomatic call site is `task \|> Task.attempt Tag` |
| `Task.executeCmd` | `Task Never (Cmd msg) -> Cmd msg`               | Run an infallible task whose success value is itself a `Cmd` (e.g. shut down a Node program after a final write) |

Use `Task.attempt` whenever the task can fail. Use `Task.perform` for an infallible task whose success value you need fed back into `update`. Use `Task.execute` for fire-and-forget infallible effects. There is **no** way to feed a fallible `Task x a` (with `x ≠ Never`) into `Task.perform` or `Task.execute` — the `Never` requirement is enforced by the type system. If you want fire-and-forget behaviour for a fallible task, use `Task.attempt` and ignore the `Err` branch in `update`.

The signatures above are why the project writes `Stream.readBytesAsString env.stdin |> Task.attempt LineRead` (callback `LineRead` applied first, then the task via `|>`) rather than the other way around.

## Example from the project

`src/Main.gren` around lines 100–108:

```gren
command =
    Cmd.batch
        [ introText
            |> Stream.Log.line env.stdout
            |> Task.execute
        , Stream.readBytesAsString env.stdin |> Task.attempt LineRead
        , Random.generate SeedGenerated (Random.int 0 1000)
        ]
```

- `Stream.Log.line stdout` builds a `Task` that writes a line; its failure type is `Never`, so it is infallible by construction. `Task.execute` runs it and we don't need its (unit) result fed back into `update`.
- `Stream.readBytesAsString stdin` builds a `Task` that reads bytes and parses them as a string. `Task.attempt LineRead` runs it and wraps the outcome in `LineRead (Result …)` so `update` can react to success or failure.
- `Random.generate` is a separate, non-`Task` helper that produces a `Cmd` directly.

## Building tasks

Common sources of `Task`:

- Stream I/O on Node: `Stream.readBytesAsString`, `Stream.Log.line`, and other `Node.*` sub-system functions (`gren-lang/node`).
- HTTP: in `gren-lang/browser`, `Http.get`/`Http.post`/`Http.request` produce a `Cmd msg` **directly** via an `Expect msg` callback — they are **not** `Task`-returning. To thread an HTTP request through `Task.andThen` chains, wrap with a task port or use the `expectString`/`expectJson` family with the result handled in `update`.
- JavaScript interop: **task ports** — see below.

You can also build tasks from pure values and chains (`gren-lang/core` 7.4.2):

| Function            | Type                                                          | Purpose                                |
| ------------------- | ------------------------------------------------------------- | -------------------------------------- |
| `Task.succeed`      | `a -> Task x a`                                               | A task that immediately succeeds       |
| `Task.fail`         | `x -> Task x a`                                               | A task that immediately fails          |
| `Task.map`          | `(a -> b) -> Task x a -> Task x b`                            | Transform success                      |
| `Task.mapError`     | `(x -> y) -> Task x a -> Task y a`                            | Transform failure                      |
| `Task.andThen`      | `(a -> Task x b) -> Task x a -> Task x b`                     | Sequence — next task depends on result. Callback is the **first** argument |
| `Task.await`        | `Task x a -> (a -> Task x b) -> Task x b`                     | Same semantics as `andThen` but with the callback **last**, so an imperative-style chain reads top-to-bottom: `Task.await firstTask <| \result -> nextTask result` |
| `Task.onError`      | `(x -> Task y a) -> Task x a -> Task y a`                     | Recover from a failure mid-chain       |

## Concurrency: parallel by default

Tasks combined with `Task.map2`, `Task.map3`, or `Task.andMap` **run concurrently by default**. From `gren-lang/core` 7.4.2's `Task.map2` docs: *"`map2` starts both requests concurrently and if either of them fails, the whole thing fails! If one request fails and the other is still in-flight, the in-flight task will be automatically cancelled and cleaned up."*

```gren
-- These two tasks run concurrently (illustrative — not via Http.get, which is Cmd-based)
Task.map2 combineResults
    taskProducingA
    taskProducingB
```

If you need **sequential** execution — A, then B — use `Task.andThen` or `Task.sequence`:

```gren
-- Sequential: A then B
taskA
    |> Task.andThen (\aResult -> taskB aResult)

-- Sequential: A, B, C in order
Task.sequence [ taskA, taskB, taskC ]
```

For an array of independent tasks you want in parallel:

```gren
Task.concurrent [ taskA, taskB, taskC ]
```

(`Task.concurrent` also cancels any in-flight tasks if one fails; see the 7.4.2 docs.)

This is Gren's semantics, and they differ from Elm, where combined tasks sequence. When a user complains that "things are running in the wrong order", this is the cause. See `09-differences-from-elm.md`.

## Ports — talking to JavaScript

Gren code cannot call JavaScript directly. To cross the boundary, you declare **ports** in your Gren module and provide handlers on the JS side. There are three styles.

### 1. Outgoing `Cmd` ports

A `Cmd` port sends a value out to JavaScript:

```gren
port sendNotification : String -> Cmd msg
```

JavaScript subscribes to it via `app.ports.sendNotification.subscribe(callback)`.

### 2. Incoming `Sub` ports

A `Sub` port receives values from JavaScript:

```gren
port clockTicks : (Int -> msg) -> Sub msg
```

JavaScript pushes values via `app.ports.clockTicks.send(42)`.

These two are the **classic** one-way port style. They work, but you end up writing two declarations per effect and managing the lifecycle on the JS side.

### 3. Task ports (preferred)

Gren adds **task ports**, which model a JS call as a `Task` — request goes out, response comes back as success or failure. This collapses the Cmd/Sub pair into one declaration:

```gren
port saveToDB : String -> Task Json.Decode.Value Int
```

The `Task err ok` form means the port takes a `String`, performs the side effect on the JS side, and returns an `Int` (or fails). On the JavaScript side you register an async function:

```javascript
Gren.Main.init({
  taskPorts: {
    saveToDB: async function (str) {
      const result = await database.save(str);
      return result.id;
    }
  }
});
```

If the async function throws, the task fails on the Gren side. If it resolves, the task succeeds with the resolved value.

Task ports are usually the cleanest option for request/response interop (database writes, calling a third-party SDK, etc.). Reserve classic Cmd/Sub ports for one-way fire-and-forget or one-way push events.

## Decoding values across ports

Ports move JSON-like values, not typed Gren data. Crossing the boundary you will use `Json.Decode` (and `Json.Encode`) to convert. This is especially true for task ports where the return type is often a `Json.Decode.Value` you then decode:

```gren
port fetchUser : String -> Task Json.Decode.Value User
```

…requires you to decode that `Value` into a `User` inside the `Task` chain. Look up the current `Json.Decode` API for the version in use.

## Mental model

- A `Task` is a *description* of an effect, like `Cmd`.
- `Task.attempt` and `Task.execute` turn a `Task` into a `Cmd` the runtime will perform.
- `Task.map2`/`map3`/`andMap` run their tasks **concurrently**; use `Task.sequence` for serial.
- Ports are the only way to call JavaScript. Task ports give you request/response semantics; classic ports give you one-way push.

## Cross-references

- How `Cmd`/`Sub` plug into `update`: `07-commands-and-subs.md`
- `Result`, which `Task` resembles: `06-errors.md`
- Gren's default task concurrency vs Elm's: `09-differences-from-elm.md`
- Concrete Node examples in the sample program: `src/Main.gren` (`init`, `respond`, `subscriptions`)
