# Viewer: views, visualizers, blueprint

The viewer is egui (immediate-mode GUI, via `eframe`) on top of wgpu (`re_renderer`), running natively and in the browser.
Each frame: query the store for the current time → each view's visualizers turn query results into draw data → render.

## Concepts

- **View class** (`re_viewer_context::ViewClass`): a kind of view, e.g. `SpatialView3D`, `TimeSeriesView`, `BarChartView`. Owns per-view state, help text, spawn heuristics (which entities get a view by default), and `ui()`.
- **Visualizer** (`VisualizerSystem` + `IdentifiedViewSystem`): turns one archetype's components for the entities in a view into something drawable. A view registers its visualizers in `ViewClass::on_register`. Examples: `Points3DVisualizer`, `BarChartVisualizerSystem`.
- **Fallback providers**: compute a component's value when it was neither logged nor overridden (registered in `on_register`, or globally in `re_component_fallbacks`).
- **Blueprint**: the viewer's own config, stored as a second store in `re_entity_db` and written with the same type system. Paths: `/viewport/`, `/container/{uuid}/`, `/view/{uuid}/`. Holds layout, visibility, per-entity overrides and view properties. Users write it from the SDK (`rerun.blueprint`) or by editing the UI.

## Smallest working example

`crates/views/re_view_bar_chart/` is the simplest built-in view (3 files):

- `src/view_class.rs`: `impl ViewClass for BarChartView`, with `on_register` doing `system_registry.register_visualizer::<BarChartVisualizerSystem>()`.
- `src/visualizer_system.rs`: `visualizer_query_info()` (which archetype it reads) and `execute()` (query, then produce output).
- `tests/bar_chart_test.rs` + `tests/snapshots/*.png`: image snapshot tests.

`examples/rust/custom_view/` builds a view and visualizer **outside** the core crates and registers it in a custom viewer binary. It is the best template for experiments that shouldn't touch core.

## Adding a view

1. New crate `crates/views/re_view_<name>/` (copy `re_view_bar_chart`); it may depend on `viewer_support` and below, never on `panels`.
2. If users should be able to pick it in a blueprint: add a blueprint view type in `crates/build/re_type_definitions/rerun/blueprint/views/` and run `pixi run codegen` (`BarChartView` uses `re_sdk_types::blueprint::views::BarChartView` as its `ViewType`).
3. Register it in `crates/top/re_viewer/src/default_views.rs` → `populate_view_class_registry_with_builtin` (`view_class_registry.add_class::<…>()`), and add the crate to `re_viewer`'s `Cargo.toml`.
4. Root `Cargo.toml` members, `pixi run crate-graph` (`CLAUDE-architecture.md`).

Adding a visualizer to an existing view is just steps 1–2 of the bar chart pattern inside that view's crate, registered in its `on_register`.

## UI code conventions

- Use `re_ui` widgets and design tokens instead of raw egui styling; see [`DESIGN.md`](DESIGN.md) for labels, casing and buttons.
- Derive ids from the parent: `ui.make_persistent_id(("plot_legend", view_id))`, never `format!`-built ids.
- Never block the UI thread; anything slow goes behind `re_tracing::profile_*!` and, if needed, off-thread (native only).

## Driving the UI (to see your change)

Two MCP servers, configured in `.mcp.json`, let an agent look at and click a running UI:

- `rerun viewer-mcp` (`crates/top/re_viewer_mcp`, binary `target/debug/re-viewer-mcp`) drives a running viewer over gRPC. Docs: `docs/content/reference/viewer/mcp.md`.
- `egui-mcp` drives any egui app started with `EGUI_INSPECTION=1`, including headless `egui_kittest` harnesses. Install with `cargo install --git https://github.com/rerun-io/kittest_inspector egui_mcp`.

Loop: start the app with `EGUI_INSPECTION=1` → `attach` (127.0.0.1:5719) → `widget_tree` → `click` / `type_text` → `screenshot` with `save_path`, then look at the image.
On macOS a windowed app must not be occluded, so prefer a headless harness, e.g. `EGUI_INSPECTION=1 cargo run -p re_agent_ui --example agent_app -- --headless`.
A test harness opts in with `egui_inspection::attach_from_env(&harness.ctx, label)`.

For checked-in verification, write an `egui_kittest` test using `re_test_context::TestContext` / `re_test_viewport` and snapshot it (`CLAUDE-build-test.md`).
