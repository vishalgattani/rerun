# Fork workflow

How this fork relates to upstream, and how to commit, sync and ship from it.

## Branches

| Branch      | Role                                                                     | Who writes it                    |
| ----------- | ------------------------------------------------------------------------ | -------------------------------- |
| `main`      | Exact mirror of `rerun-io/rerun` `main`. Never commit here.               | The sync workflow only           |
| `develop`   | Default branch. Our features, on top of upstream.                        | Us                               |
| `feat/<slug>` | Optional short-lived branch per feature, merged back into `develop`.   | Us                               |

The only fork-specific files on `develop` are `CLAUDE.md`, the `CLAUDE-*.md` topic files, and `.github/workflows/sync-upstream.yml`.
Upstream ships `CLAUDE.md` as a symlink to `AGENTS.md`; on `develop` it is a real file, and `AGENTS.md` is left untouched.

## Syncing upstream

`.github/workflows/sync-upstream.yml` (on `develop`, because scheduled workflows only run from the default branch) fast-forwards `main` to upstream `main` daily at 06:00 UTC via the GitHub `merge-upstream` API.
It authenticates with the `SYNC_TOKEN` repo secret (a PAT with `repo` + `workflow`), falling back to `GITHUB_TOKEN`, which cannot push upstream changes to `.github/workflows/`.
Run it on demand:

```sh
gh workflow run sync-upstream.yml -R vishalgattani/rerun --ref develop
```

`develop` is **not** updated automatically. To bring upstream into it:

```sh
git fetch origin
git switch develop
git merge origin/main          # merge, don't rebase: develop is shared and pushed
git diff ORIG_HEAD..HEAD -- AGENTS.md   # did upstream's agent guidance change? port it into the CLAUDE-*.md files
```

Conflicts are most likely in `Cargo.lock`, `pixi.lock`, `uv.lock`, and generated code.
For lock files, take upstream's version and re-resolve (`cargo update -w`, `pixi install`).
For generated code, take either side and re-run `pixi run codegen`.

## Upstream CI in the fork

Upstream's workflows live in `.github/workflows/` and are not restricted to `rerun-io/rerun`, so they can run in this fork:
`nightly.yml` is scheduled, `on_push_main.yml` and `cargo_shear.yml` run on pushes to `main` (which the sync makes with a PAT), and the `pull_request*` ones run on PRs.
Most need rerun-io secrets and runners and will fail here.
Disable the ones we don't want with `gh workflow disable <file> -R vishalgattani/rerun` rather than editing upstream's workflow files, which would conflict on every sync.

## Commits, pushes, PRs

- Git identity is `digif33ls <digif33ls@gmail.com>` (set in this clone's `.git/config`, along with `credential.tokenEnv`).
- Commit messages: `feat|fix|chore|docs(<crate-or-area>): <summary>`, e.g. `feat(re_view_bar_chart): stacked bars`.
- Push only when Vishal says "push" in that turn.
- PRs inside the fork (`feat/*` → `develop`): `gh pr create --draft -R vishalgattani/rerun --base develop`. Vishal merges.
- **Never open a PR, issue or comment on `rerun-io/rerun` unless asked.** If asked, follow upstream's rules in `AGENTS.md` → "Contributing" (PR template, LLM disclosure, changelog label, one `docs/content/changelog/upcoming/<slug>.md` entry for user-facing changes), and branch from `main`, not `develop`, so the PR contains no fork-only files.

## Keeping features upstreamable

Keep each feature a self-contained set of commits on top of upstream, following upstream's style (`CLAUDE-style.md`), so it can be cherry-picked onto a branch off `main` for an upstream PR later.
Prefer extension points that need no core changes when they fit: a custom view or visualizer (`examples/rust/custom_view`), a custom importer (`examples/rust/custom_importer`, or an external `rerun-importer-*` executable), or a blueprint built from the SDK.
