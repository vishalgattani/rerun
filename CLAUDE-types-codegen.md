# Types and code generation

**Never edit generated files** (header says `DO NOT EDIT`): Rust in `crates/store/re_sdk_types/src/`, Python in `rerun_py/rerun_sdk/rerun/{archetypes,components,datatypes,blueprint}/`, C++ in `rerun_cpp/src/rerun/`, docs in `docs/content/reference/types/`.

## The type hierarchy

1. **Datatypes / encodings** (`rerun.datatypes.*`, `rerun.encodings.*`): plain data, e.g. `Vec3D`, `Rgba32`.
2. **Components** (`rerun.components.*`): a semantic name over a datatype, e.g. `Position3D`, `Radius`, `Color`.
3. **Archetypes** (`rerun.archetypes.*`): what users log, a bundle of components, e.g. `Points3D`, `Image`, `BarChart`.

Each archetype field is `#[rerun(required)]`, `#[rerun(recommended)]` or `#[rerun(optional)]`.
Blueprint types (`rerun.blueprint.*`: archetypes, components, `views`) configure the viewer and use the same machinery.

## Flow

```
crates/build/re_type_definitions/rerun/{datatypes,encodings,components,archetypes,blueprint,testing}/*.def.rs
   → pixi run codegen     (crates/build/re_types_builder)
   → Rust + Python + C++ + reference docs
```

Definitions are a Rust subset, annotated with the `#[rerun::rerun_type]` macro (`re_types_builder_macros`).
See `archetypes/bar_chart.def.rs` for a compact example: docstring (becomes docs in every language), `\example archetypes/<snippet>`, `#[docs(category, view_types)]`, `#[rerun(state = "stable")]`, `#[rerun(visualizer = "BarChart")]`, and per-field attributes.
New definition files must also be listed in the sibling module file (`archetypes.rs`, `components.rs`, …).

## Extending generated types

Hand-written additions go in an `_ext` file next to the generated one, picked up automatically by codegen:

| Language | File                    | Use for |
| -------- | ----------------------- | ------- |
| Rust     | `<name>_ext.rs`         | extra constructors, `impl From`, helper methods |
| Python   | `<name>_ext.py`         | custom `__init__`, converters, numpy handling (mixed into the generated class) |
| C++      | `<name>_ext.cpp`        | extra methods; marked sections get copied into the generated header |

## Checklist: new archetype end to end

1. Components first: reuse existing ones; add `components/<name>.def.rs` only for a genuinely new semantic.
2. `archetypes/<name>.def.rs` + register it in `archetypes.rs`. Set `state` (start with `"unstable"`), docs category and `view_types`.
3. `pixi run codegen`, then `pixi run rs-fmt`. Commit definition and generated output together.
4. Add `_ext` files for ergonomic constructors where the generated API is clumsy.
5. A snippet in Python, Rust and C++ under `docs/snippets/all/archetypes/<name>.*`, referenced by `\example` (see `docs/snippets/README.md`, `snippets.toml`).
6. Make the viewer show it: a visualizer, usually in an existing view (`CLAUDE-viewer.md`). Point `#[rerun(visualizer = …)]` at it.
7. Component UIs/fallbacks if needed: `re_component_ui` (editors), `re_component_fallbacks` (defaults).
8. Tests: Python round-trip in `rerun_py/tests/`, Rust in the visualizer's view crate (image snapshot).
9. A changelog entry `docs/content/changelog/upcoming/<slug>.md` if it is user-facing (`CLAUDE-style.md`).

Protobuf types (`crates/store/re_protos/proto/`) are a separate generator: `pixi run codegen-protos`, checked with `pixi run pb-lint` and `pb-breaking`.
