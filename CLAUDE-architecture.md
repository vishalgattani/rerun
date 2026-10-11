# Architecture

Upstream reference: [`ARCHITECTURE.md`](ARCHITECTURE.md) (one-line description of every crate) and [`crates/README.md`](crates/README.md) (layer rules).

## Data flow

```
SDK (rr.log(entity_path, Archetype))             crates/top/re_sdk, rerun_py, rerun_cpp, rerun_js
  → Chunk: Arrow columns + timelines             crates/store/re_chunk
  → LogMsg over gRPC / .rrd file / memory        re_log_msg, re_log_encoding, data_flow/re_grpc_*
  → ChunkStore, indexed per entity + timeline    crates/store/re_chunk_store
  → EntityDb (recording + blueprint stores)      crates/store_app/re_entity_db
  → queries (latest-at, range, dataframe)        re_query, re_dataframe, re_datafusion
  → views + visualizers, every frame             crates/views/*, viewer_support/re_view*
  → re_renderer (wgpu) + egui                    viewer_support/re_renderer, re_ui
```

Everything is Apache Arrow end to end: stored, sent and queried as Arrow arrays.
An `.rrd` file is a stream of `LogMsg`s; the viewer opens files from the previous version, older ones are not guaranteed.

## Crate layers

Each folder under `crates/` is a layer; a crate may depend on its own folder or any **lower** one, never higher (`dev-dependencies` exempt).
`scripts/check_crate_layers.py` enforces it in CI. Top to bottom:

| Layer            | Holds                                                                 | Examples |
| ---------------- | --------------------------------------------------------------------- | -------- |
| `tests`          | Test-only support                                                     | `re_test_context`, `re_test_viewport` |
| `top`            | SDKs, bindings, CLI, the viewer app                                   | `re_sdk`, `rerun`, `rerun-cli`, `re_viewer`, `rerun_c` (+ `rerun_py/`) |
| `views` ∥ `panels` | Siblings, neither depends on the other. Views go in the viewport; panels are the app around it | `re_view_spatial`, `re_view_time_series` ∥ `re_viewport`, `re_time_panel`, `re_selection_panel` |
| `viewer_support` | UI and rendering machinery any view/panel may use                     | `re_viewer_context`, `re_view`, `re_ui`, `re_renderer`, `re_data_ui`, `re_component_ui` |
| `store_app`      | Queryable state for a viewer or server                                | `re_entity_db`, `re_query`, `re_dataframe`, `re_server`, `re_tf` |
| `data_flow`      | Moving data in/out: gRPC, file importers                              | `re_importer`, `re_data_source`, `re_mcap`, `re_parquet`, `re_grpc_client` |
| `store`          | Data model, chunks, store, protobufs                                  | `re_chunk`, `re_chunk_store`, `re_sdk_types`, `re_types_core`, `re_log_types` |
| `build`          | Build-time only, never shipped                                        | `re_type_definitions`, `re_types_builder` |
| `utils`          | Small helpers with no Rerun deps outside `utils`                      | `re_log`, `re_tracing`, `re_format`, `re_video` |

## Adding a crate

1. Pick the lowest layer whose description fits; if it then needs something higher, it belongs higher (or that thing belongs lower). Fix a wrong-way dependency by moving a crate, never by reordering layers.
2. Add it to `members` in the root `Cargo.toml`; give its `Cargo.toml` a one-line `description` (it feeds the docs tables).
3. Run `pixi run crate-graph` to regenerate `crate_graph.svg` and the tables in `ARCHITECTURE.md` (CI does not check this).
4. Run `pixi run hakari-generate` if it adds third-party dependencies (`re_workspace_hack` unifies features).

## Where to look for…

| Concern                                  | Start in |
| ---------------------------------------- | -------- |
| App shell, menus, startup, view registration | `crates/top/re_viewer/src/` (`app/`, `app_state.rs`, `default_views.rs`) |
| Shared per-frame viewer state, view/visualizer traits | `crates/viewer_support/re_viewer_context/` |
| Blueprint data model (layout, containers, overrides) | `crates/viewer_support/re_viewport_blueprint/` |
| Transforms and coordinate frames         | `crates/store_app/re_tf/` |
| CLI subcommands and flags                | `crates/top/rerun/src/commands/` |
| Python ↔ Rust bindings                   | `rerun_py/src/` (PyO3, module `rerun_bindings`) |
| Protobuf / gRPC API                      | `crates/store/re_protos/proto/` (regenerate with `pixi run codegen-protos`) |
| Viewer MCP server for agents             | `crates/top/re_viewer_mcp/` |

## Constraints that shape every change

- **Immediate mode.** Each frame queries the store and redraws everything; there is no retained UI state or change callbacks. Keep per-frame work cheap and add `re_tracing::profile_function!()` to anything that may be slow.
- **Wasm.** The viewer also compiles to `wasm32`: no threads, no file system, no env vars, no `std::time::Instant` (use `web-time`/`instant`), no raw networking. Prefer a runtime check (e.g. disabled button on web) over `#[cfg]` in UI code.
- **Multi-language.** A type-definition change regenerates Rust, Python, C++ and docs together (`CLAUDE-types-codegen.md`).
