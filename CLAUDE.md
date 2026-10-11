# CLAUDE.md

Guidance for Claude Code in **`vishalgattani/rerun`**, a fork of [`rerun-io/rerun`](https://github.com/rerun-io/rerun) used to build features on `develop`.

Rerun is a time-aware multimodal data stack: SDKs (Python, Rust, C++, JS) log Arrow-encoded data, and a Viewer (native + Wasm, egui + wgpu) visualizes it.
It is a large Rust workspace (~120 crates under `crates/`), managed with `pixi`.

This file is deliberately short.
Read the topic file that matches the task **before** starting it:

| When the task touches…                                                  | Read                                                   |
| ----------------------------------------------------------------------- | ------------------------------------------------------ |
| Branches, syncing upstream, commits, pushing, PRs                       | [`CLAUDE-fork.md`](CLAUDE-fork.md)                     |
| Building, running, testing, snapshots, linting, toolchain setup         | [`CLAUDE-build-test.md`](CLAUDE-build-test.md)         |
| Where code lives, crate layers, data flow, adding a crate               | [`CLAUDE-architecture.md`](CLAUDE-architecture.md)     |
| A new or changed archetype / component / datatype, codegen, `_ext` files | [`CLAUDE-types-codegen.md`](CLAUDE-types-codegen.md)   |
| Viewer UI: views, visualizers, blueprint, panels, driving the UI        | [`CLAUDE-viewer.md`](CLAUDE-viewer.md)                 |
| Getting data in: importers, file formats, SDK APIs (Py/Rust/C++/JS)     | [`CLAUDE-sdks-importers.md`](CLAUDE-sdks-importers.md) |
| Code style, comments, docs, docstrings, changelog entries               | [`CLAUDE-style.md`](CLAUDE-style.md)                   |

[`AGENTS.md`](AGENTS.md) is upstream's own agent guide, kept unmodified so syncs stay conflict-free.
The topic files above cover it; consult it directly only when they don't.

## Rules that always apply

- **Commit and push only to `develop` in `vishalgattani/rerun`.** No other branch (`main` is a pure mirror of upstream), and never anything to `rerun-io/rerun`: no pushes, PRs, issues or comments.
- Never edit generated files (marked `DO NOT EDIT`). Change `crates/build/re_type_definitions/` and run `pixi run codegen`.
- A crate may only depend on its own `crates/<layer>/` folder or a lower one — see `CLAUDE-architecture.md`.
- Never panic in non-test code: no `unwrap`/`expect`/slice indexing on fallible paths. Errors are returned or logged, never ignored.
- The viewer is immediate mode: every frame re-queries the store and redraws. No callbacks, no cached UI state.
- Wasm builds must not spawn threads or touch files, env vars, `std::time`, or raw sockets.
- After Rust changes run `pixi run rs-fmt`, then clippy and nextest on the crates you touched.
- Commit as `digif33ls`. Push only when Vishal says "push" in that turn.

## Quick commands

```sh
pixi run rerun                                    # run the native viewer (debug)
pixi run py-build && pixi run uvpy script.py      # build Python SDK, run a script with it
cargo clippy -p <crate> --all-features
cargo nextest run -p <crate> --all-features --no-fail-fast
pixi run codegen                                  # after editing type definitions
pixi run rs-fmt                                   # always, after Rust edits
```
