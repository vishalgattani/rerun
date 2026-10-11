# Style, docs, changelog

Full rules: [`CODE_STYLE.md`](CODE_STYLE.md) (code) and [`DESIGN.md`](DESIGN.md) (prose, UI labels). The ones that matter most:

## Rust

- No panics outside tests: replace `unwrap`/`expect`/`v[i]` with `let Some(x) = v.get(i) else { … };`. `unsafe` only when unavoidable.
- Errors: `thiserror` in libraries (never `anyhow` in a library crate; add a variant), `anyhow` in apps. Never swallow an error: return it or log it (`re_log::error!/warn!` or their `_once` variants). Display errors with `{err:#}` or `re_error::format(err)`, strings with `{s:?}`.
- In messages, the error comes first and any file path last: `Failed to import: {err}\nFile path: {path}`.
- `format!("{x}")` inline args; numbers, sizes and durations shown to humans go through `re_format`.
- Hash maps: `ahash::HashMap`; `nohash_hasher::IntMap/IntSet` for keys that are already hashes (`EntityPath`, `ComponentDescriptor`, …). Sort anything returned from a hash map, or put `unsorted` in the name.
- `use Trait as _;` for trait-method imports; `foo().ok();` over `let _ = foo();`; destructure `let Self { a, b } = self;` when using most fields.
- No catch-all `_ =>` on our own enums. Keep hand-written lists sorted.
- `cfg_select!` over paired `#[cfg]`/`#[cfg(not)]`.
- Constants for every threshold and timeout, with where the number comes from. Metadata keys and column names through their constants, never string literals.
- API: private by default, constructors over free functions, no global state, nothing test-only in production code, no code "for later".
- `lib.rs`/`mod.rs` hold only `mod` and `pub use` lines.
- Before writing a helper, look for one here or in egui/emath/quiver.
- Docstrings on every public item: what it does, units, defaults and edge cases, not how. Comments only for non-obvious invariants, never narration or history.
- A to-do marker without an owner fails CI: give it an owner in parentheses (`vishal`) or link an issue.

## Python

Keyword-only args for non-obvious parameters; ruff formatting (`pixi run py-fmt`); mypy-clean.
Docstrings are **markdown for MkDocs + mkdocstrings, never reStructuredText**:
cross-refs `[`ClassName`][]`, admonitions `!!! warning`, fenced code blocks, numpy-style `Parameters` / `Returns` sections, deprecations via the `@deprecated` decorator only.

## Prose (docs, comments, UI, log messages)

- `…` instead of three dots; spaced em dash ` — `, never unspaced; `–` only for numeric ranges. `pixi run lint-rerun` checks these.
- One sentence per line in markdown.
- Sentence case for labels and headings; see `DESIGN.md`.

## Docs

- Main site source: `docs/content/` (see `docs/README.md`). API docs: Python (MkDocs, `pixi run py-docs-serve`), C++ (Doxygen), JS (TypeDoc).
- Generated, don't edit: `docs/content/reference/types/` (codegen) and `docs/content/reference/cli.md` (`pixi run man`).

## Changelog

Upstream generates the changelog from PR labels plus hand-written entries.
For a user-facing feature, copy `docs/content/changelog/upcoming/_template.md` to `upcoming/<short-slug>.md` and fill it in (`type: highlight|misc|breaking|feature`).
It costs nothing on `develop` and makes an upstream PR ready later.
