# SDKs and importers

## SDKs

| SDK    | Lives in                                         | Notes |
| ------ | ------------------------------------------------ | ----- |
| Rust   | `crates/top/re_sdk` (core), `crates/top/rerun` (public crate, re-exports + CLI commands) | `RecordingStream` is the logging entry point |
| Python | `rerun_py/rerun_sdk/rerun/` (Python), `rerun_py/src/` (PyO3 → `rerun_bindings`) | Built with maturin via `pixi run py-build`; see `rerun_py/README.md` |
| C++    | `rerun_cpp/` on top of `crates/top/rerun_c` (C API) | CMake; `pixi run -e cpp …` |
| JS     | `rerun_js/web-viewer`, `rerun_js/web-viewer-react` | Embeds the Wasm viewer; `pixi run js-build-all` |

Most of each SDK's archetype/component API is generated (`CLAUDE-types-codegen.md`). Hand-written surface:
the recording stream (`recording_stream.py`, `re_sdk`), `send_columns` / `send_dataframe`, blueprints (`rerun/blueprint/`), the catalog client (`rerun/catalog/`, `rerun_py/src/catalog/`), and `_ext` files.

Python API rules: keyword-only arguments for anything non-obvious (`def serve(*, ip: str, port: int = 80)`), numpy-style docstrings in markdown for MkDocs (`CLAUDE-style.md`).
Changing a public Python signature: run `pixi run py-check-signatures`.

## Examples and snippets

- `examples/{python,rust,cpp,c}/`: full examples. Python ones each have a `pyproject.toml`; Rust ones are workspace packages (`cargo run -p <name>`). Listed in `examples/manifest.toml`.
- `docs/snippets/all/<area>/<name>.{py,rs,cpp}`: short snippets used in docs and cross-language comparison tests. See `docs/snippets/README.md`.

## Importers

`crates/data_flow/re_importer` turns files into chunks for `rerun <file>`, drag-and-drop, and `rr.log_file_from_path`.

- Trait `re_importer::Importer`: `name()`, `import_from_path()` (native only), `import_from_file_contents()`. Return `ImporterError::Incompatible` fast for files you don't handle.
- Built-ins (`BUILTIN_IMPORTERS` in `src/lib.rs`): rrd, archetype (images, meshes, point clouds, video, …), directory, MCAP, parquet, LeRobot, external, puffin, URDF.
- Format-specific logic lives in its own `data_flow` crate (`re_mcap`, `re_parquet`, `re_lerobot`, `re_mp4_reader`, `re_hdf5`) with a thin adapter in `re_importer/src/importer_*`.
- Three ways to add a format, cheapest first:
  1. **External importer**: any executable on `PATH` named `rerun-importer-<name>` that writes an RRD stream to stdout. No Rerun changes at all. See `examples/python/external_importer` and `examples/rust/external_importer`.
  2. **Custom importer in your own binary**: `re_importer::register_custom_importer(…)`; see `examples/rust/custom_importer`.
  3. **Built-in**: a new `importer_<fmt>.rs` (plus a `data_flow` crate if non-trivial) added to `BUILTIN_IMPORTERS`, behind a Cargo feature if it pulls heavy dependencies.

The repo's `skills/` directory (upstream's user-facing agent skills: `rerun-data-model`, `rerun-mcap`, `rerun-parquet`, `rerun-urdf`, `rerun-blueprint`, …) documents how to model and ingest data with the SDK. Read `skills/rerun-data-model/SKILL.md` before designing how a new dataset maps onto entities and components.
