# Fork workflow

How this fork relates to upstream, and how to commit, sync and ship from it.

## Branches

| Branch    | Role                                                      | Who writes it          |
| --------- | --------------------------------------------------------- | ---------------------- |
| `main`    | Exact mirror of `rerun-io/rerun` `main`. Never commit here. | The sync workflow only |
| `develop` | Default branch. All our work, on top of upstream.          | Us — the only branch we commit or push to |

**Never create, commit to, or push any other branch, and never write to `rerun-io/rerun`** (no pushes, PRs, issues or comments).

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
`nightly.yml`, `on_push_main.yml` and `cargo_shear.yml` are disabled in the fork's Actions settings (`gh workflow list -R vishalgattani/rerun --all`).
Disable others the same way (`gh workflow disable <file> -R vishalgattani/rerun`) rather than editing upstream's workflow files, which would conflict on every sync.

## Commits, pushes, PRs

- Git identity is `digif33ls <digif33ls@gmail.com>` (set in this clone's `.git/config`, along with `credential.tokenEnv`).
- Commit messages: `feat|fix|chore|docs(<crate-or-area>): <summary>`, e.g. `feat(re_view_bar_chart): stacked bars`.
- Commit directly on `develop`, then push `develop` only when Vishal says "push" in that turn: `git push origin develop`.
- No feature branches and no PRs, in the fork or upstream.
- Ignore the PR, labelling and review instructions in `AGENTS.md` → "Contributing"; they are for upstream contributors.

## Building features

Keep each feature a self-contained set of commits on `develop` that follows upstream's style (`CLAUDE-style.md`), so upstream merges into `develop` stay easy.
Prefer extension points that need no core changes when they fit: a custom view or visualizer (`examples/rust/custom_view`), a custom importer (`examples/rust/custom_importer`, or an external `rerun-importer-*` executable), or a blueprint built from the SDK.
