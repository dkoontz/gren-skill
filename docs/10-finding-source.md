# 10 — Finding Source and API Information

The Gren Coach never guesses at APIs. When you don't have a function memorised, this is the workflow for finding out. Use this doc as your reference whenever a question lands on a specific library or module.

## The official sources

| Need                                            | Where                                                                                |
| ----------------------------------------------- | ------------------------------------------------------------------------------------- |
| Language guide (concepts)                        | <https://gren-lang.org/book/>                                                          |
| Compiler and tooling (source)                    | <https://github.com/gren-lang>                                                          |
| Core standard library (source + docs)            | <https://github.com/gren-lang/core>                                                     |
| Node runtime package                             | <https://github.com/gren-lang/node>                                                     |
| Browser runtime package                          | <https://github.com/gren-lang/browser>                                                  |
| Published package catalogue (every published pkg) | <https://packages.gren-lang.org/package>                                                |
| Specific package overview by version             | `https://packages.gren-lang.org/package/<author>/<package>/version/<version>/`         |
| Specific module docs by version                  | `https://packages.gren-lang.org/package/<author>/<package>/version/<version>/module/<Module>` |

The compiler series this doc set was verified against is **0.6.5**. The project you are coaching may use a different version — always check `gren.json`'s `gren-version` field and read the docs for that version.

## Where source lives in a Gren project

A typical Gren project layout:

```
.
├── gren.json                ← project manifest (deps, source-directories, version)
├── gren_packages/           ← tarballs of dependencies (downloaded; do not edit)
│   ├── gren_lang_core__<v>.pkg.gz
│   ├── gren_lang_node__<v>.pkg.gz   (if Node platform)
│   └── ...
├── .gren/
│   └── <version>/           ← compiler binaries for that version
├── src/                     ← the project's own source
│   ├── Main.gren
│   └── ...                  (run.js for Node projects; index.html etc. for browser)
└── build/                   ← compiler output (if it has been built)
```

- `gren.json` is the manifest. Read it to learn **what packages this project depends on** and **which directories are part of the project source**. The `source-directories` field tells the compiler where to look for the project's own modules. The `platform` field is either `"node"` or `"browser"`.
- `gren_packages/` holds downloaded tarballs. They are gzipped — you cannot read them directly without extracting. For browsing source, prefer the GitHub repositories below over the local tarballs.

## How to look up a function in a Gren project

1. **Identify the module.** Read the `import` lines at the top of the relevant `.gren` file (often `src/Main.gren`). Each imported module belongs to exactly one package.

2. **Identify the package.** Cross-reference the module name against `gren.json`'s `dependencies`:
   - `Array`, `Bytes`, `Random`, `Result`, `Maybe`, `Task`, `String`, `Dict`, `Set`, `Json.*`, `Math`, `Basics`, `Char`, `Time` → `gren-lang/core`
   - `Node`, `Node.*`, `Stream`, `Stream.Log`, `Init`, `FileSystem`, `Terminal`, … → `gren-lang/node`
   - `Browser`, `Html`, `Html.Attributes`, `Html.Events`, `Http`, `File`, … → `gren-lang/browser`
   - `Url`, `Url.*`, `Url.Builder`, `Url.Parser` → `gren-lang/url`
   - Any third-party module → its own package (see `gren.json`'s `dependencies.direct` and `dependencies.indirect`)

3. **Read the package's docs at the pinned version.** The version lives in `gren.json` — for example, if `gren-lang/core` is pinned to `7.4.2`, browse to:
   ```
   https://packages.gren-lang.org/package/gren-lang/core/version/7.4.2/
   ```
   Add `/module/<ModuleName>` to deep-link a specific module, e.g. `/version/7.4.2/module/Array`.

4. **Or read the source directly** on GitHub. The latest `main` may be ahead of the pinned version, so prefer the published docs for the pinned version when you need exact behaviour. If GitHub is easier, look at the `README.md` in the repo and the per-module files under `src/`.

## What to do when the Coach doesn't know

- Be honest: "I'm not sure if `Array` has a function for X — let me check."
- Use the web tools (`webfetch`, `websearch`) to fetch the package page or search for the function.
- If the user names a function, grep the project for it: `rg "theFunctionName" .` — sometimes the project already imports and uses it, which is the most reliable local evidence of its existence.
- If you cannot verify, **say so**. Suggest the user run `gren repl` and try the function interactively, or look it up on the package site. Never invent a function.

## Useful read-only shell commands

```sh
# Show project dependencies, platform, and source dirs
cat gren.json

# List project source files
fd -e gren . src

# Find every type annotation in the project
rg "^.* : " src

# Find every custom type or type alias
rg "^(type|type alias) " src

# Find where a given function is used
rg "\bparseCommand\b" src

# See what modules exist in a package (after extracting, or on GitHub)
# e.g. https://github.com/gren-lang/core/tree/main/src
```

Do **not** run mutating commands (`gren make`, `gren package install`, etc.) — those belong to other agents. The Coach is read-only.

## Looking at dependency source on GitHub

When you need to know exactly how a function behaves, the source is the truth:

- `gren-lang/core` modules live under `src/` in the repo. Module `Array` is in `src/Array.gren`, etc.
- `gren-lang/node` provides `Node`, `Stream`, `Stream.Log`, `Init`, `FileSystem`, `Terminal`, and other runtime modules. The full list changes between versions — see the repo's `src/` listing or the package docs.
- `gren-lang/browser` provides `Browser`, `Html`, `Html.Attributes`, `Html.Events`, `Http`, and friends. Same caveat — verify against the pinned version.

To deep-link to a function: `https://github.com/gren-lang/core/blob/main/src/Array.gren` and find the function by scrolling. Pin to a tag matching the version in `gren.json` if you need to be precise.

## Cross-references

- What you'll typically find in each module: `04-data-structures.md` (Array/Dict/Set), `06-errors.md` (Maybe/Result), `07-commands-and-subs.md` (Node/Browser), `08-task-ports.md` (Task/ports).
- The project's own imports and module list: top of its main `.gren` file (often `src/Main.gren`).
