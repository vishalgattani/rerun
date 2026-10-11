# Build, run, test

Full details: [`BUILD.md`](BUILD.md), [`TESTING.md`](TESTING.md), [`CONTRIBUTING.md`](CONTRIBUTING.md) → "Tests". All tasks: `pixi task list` / `pixi.toml`.

## One-time setup (this Mac has none of it yet)

```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh   # rustup; rust-toolchain pins 1.96.0 + wasm32 target
curl -fsSL https://pixi.sh/install.sh | sh                       # pixi >= 0.78 (provides uv, nextest, protoc, cmake, … per pixi.toml)
brew install git-lfs && git lfs install && git lfs pull          # snapshot PNGs are LFS: without this they are pointer files and image tests fail
pixi run check-env                                               # validate
```

On Apple silicon, `rustc -vV` must report `host: aarch64-apple-darwin`.
If `cargo` errors with `failed to parse contents of PYO3_CONFIG_FILE`, the generated `rerun_py/pyo3-build.cfg` is missing.
Pixi activation (`scripts/pixi/activate.sh` → `ensure-rerun-env`) writes it, so run `pixi run ensure-rerun-env` once, or use `pixi run`/`pixi shell` rather than bare `cargo` on a fresh clone.
`BUILD.md` and `AGENTS.md` still say `pixi run ensure-pyo3-build-cfg`; that task no longer exists.

## Build and run

| Goal                                   | Command                                             |
| -------------------------------------- | --------------------------------------------------- |
| Native viewer, debug                   | `pixi run rerun` (args after `--`, e.g. `pixi run rerun file.rrd`) |
| Native viewer, release                 | `pixi run rerun-release`                            |
| Fast iteration build (nightly, `dev-fast` profile) | `pixi run rerun-build-fast` then `pixi run rerun-fast` |
| Web viewer                             | `pixi run rerun-web` (build only: `pixi run rerun-build-web`) |
| Python SDK into `.venv` (maturin, uv)  | `pixi run py-build`                                 |
| Run a Python script against it         | `pixi run uvpy script.py`                           |
| Rust example                           | `cargo run -p <example>` (e.g. `cargo run -p dna`)  |
| C++ SDK + tests                        | `pixi run -e cpp cpp-build-all`, `pixi run -e cpp cpp-test` |
| JS packages                            | `pixi run js-build-all`                             |

Python uses its own uv-managed `.venv`, not pixi's conda env; the `uvpy` wrapper unsets `CONDA_PREFIX` to keep them apart.

## Test

```sh
cargo clippy -p <crate> --all-features
cargo nextest run -p <crate> --all-features --no-fail-fast    # nextest, not cargo test; --all-features unless there's a reason
pixi run py-test                                              # pytest (+ syrupy snapshots); builds the SDK first
pixi run -e cpp cpp-test
```

- **Text snapshots** (`insta`): update with `INSTA_FORCE_PASS=1 cargo nextest run -p <crate> --all-features`, then `cargo insta review`.
- **UI image snapshots** (`egui_kittest` `Harness::snapshot`, `re_test_context::TestContext` mocks the viewer): stored in `<crate>/tests/snapshots/`, failures write `*.diff.png`. Update with `UPDATE_SNAPSHOTS=1 cargo nextest run -p <crate> --all-features`. Snapshots differ per OS (`*_mac.png`, `*_windows.png`); only update the ones for the platform you ran on.
- **Python snapshots**: add `--snapshot-update` to the pytest command.
- **Server / protocol**: `cargo nextest run -p re_server --all-features` runs the `re_redap_tests` compliance suite.

Test behavior that can plausibly break, one behavior per test, through the UI rather than internal state, with small fixtures and exact asserts.

## Lint and format (what CI checks)

| Area     | Fix                     | Check                                   |
| -------- | ----------------------- | --------------------------------------- |
| Rust     | `pixi run rs-fmt`       | `pixi run rs-check` (clippy matrix, slow) |
| Python   | `pixi run py-fmt`       | `pixi run py-lint-rerun` (mypy)         |
| C++      | `pixi run cpp-fmt`      |                                         |
| TOML     | `pixi run toml-fmt`     | `pixi run toml-fmt-check`               |
| YAML/JS  | `pixi run misc-fmt`     |                                         |
| Codegen  | `pixi run codegen`      | `pixi run lint-codegen`                 |
| Rerun conventions (`…`, dashes, owner-less to-dos, …) | | `pixi run lint-rerun [<file>]` |
| Spelling |                         | `pixi run lint-typos`                   |
| Crate layering |                   | `python scripts/check_crate_layers.py`  |

## Before calling a change done

1. `pixi run rs-fmt` (and `py-fmt` / `cpp-fmt` if touched).
2. `pixi run codegen` if any type definition changed; commit the regenerated files with it.
3. clippy + nextest on every crate touched; `py-test` if the Python SDK changed.
4. `pixi run lint-rerun` on changed files.
5. For UI changes, look at the result (see `CLAUDE-viewer.md` → "Driving the UI") and update snapshots.
