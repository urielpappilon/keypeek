# Fork workflow

This fork lives at `urielpappilon/keypeek`. Default branch is `main`. `master` is *not* maintained on the fork — local `master` is a read-only mirror of upstream `srwi/keypeek@master` and is never pushed to `origin`.

## Remotes

```
upstream  →  srwi/keypeek          (read-only; sync source)
origin    →  urielpappilon/keypeek (read/write; the fork)
```

Set up once on a fresh clone:

```sh
git remote rename origin upstream  # if you cloned from srwi/keypeek
git remote add origin git@github.com:urielpappilon/keypeek.git
git config rerere.enabled true     # remember conflict resolutions across rebases
git fetch --all
git checkout main
```

## Branch layout

### `main` — the integration branch

Linear history: `upstream/master` + every adopted upstream-PR commit, cherry-picked in topological order. This is what gets built and run. Push to `origin/main`.

### `pr<N>/<short-name>` — atomic topic branches

Each is the *smallest* sufficient cherry-pick of an upstream PR's commits, branched from either `upstream/master` directly (if the commit applies cleanly) or from another topic branch it logically depends on. These exist so individual changes can be sent back upstream as clean, single-purpose PRs if/when maintainers reopen.

Current set:

| Branch | Base | Source | Notes |
|---|---|---|---|
| `pr15/qmk-mod-shortlabels` | `upstream/master` | PR #15 c1 | Text-style QMK mod labels (`LSft`/`LCtl`/etc) + bit-decode fix |
| `pr15/multiline-key-labels` | `upstream/master` | PR #15 c2 | Multi-line label rendering (only useful with `\n`-bearing labels) |
| `pr17/cli-toggle-and-ipc` | `upstream/master` | PR #17 `f168696` | CLI scaffolding, `--toggle`, IPC socket, autoconnect, settings persistence |
| `pr17/cli-settings-command` | `pr17/cli-toggle-and-ipc` | `c152c95` + `c9e5a21` | `--settings` command |
| `pr17/tap-hold-bottom-rendering` | `upstream/master` | `7c7be9c` | Hold label rendered at bottom of key |
| `pr17/mouse-icons` | `upstream/master` | `658d956` + `98c213a` | Icons for mouse buttons + direction/scroll |
| `pr17/active-layer-base-color` | `upstream/master` | `8bafaf8` | Base color by active layer, not first |

The remaining PR #17 commits (modifier-symbol-rendering, shifted-key-rendering, dedup+tap-dance, split-keyboard layout fix, short-keypress highlight fix, HID error handling, stale-socket recovery, detach-toggle, code-rabbit cleanups) have too tangled a dependency chain to live as independent topic branches without rewriting code. They exist on `main` as their original commits — extract them with `git cherry-pick` from `main` onto a fresh `upstream/master` branch when needed for upstream.

### `compare/pr15`

`upstream/master` + the two PR #15 commits, no PR #17. Built into `KeyPeek-pr15.app` so you can compare PR #15's text-mod-label rendering against `main`'s symbol rendering before deciding whether to invest in a `main-text` variant.

## Common workflows

### Sync upstream into the fork

```sh
git fetch upstream
git checkout master
git merge --ff-only upstream/master           # local mirror, NEVER push
git checkout main
git rebase upstream/master                    # rerere replays any prior resolutions
# Resolve any new conflicts. Then:
git push --force-with-lease origin main
```

For each topic branch:

```sh
git checkout pr17/<name>
git rebase upstream/master                    # or rebase --onto parent-topic if it stacks
git push --force-with-lease origin pr17/<name>
```

### Drop a topic branch when upstream merges its equivalent

After upstream merges the equivalent change, the commits will appear in `upstream/master`. Then:

1. Identify the topic branch's commits by SHA.
2. Rebase `main` onto the new `upstream/master`. Git will skip already-applied commits.
3. Delete the topic branch:
   ```sh
   git branch -D pr17/<name>
   git push origin --delete pr17/<name>
   ```

### Submit a topic branch upstream

```sh
git checkout pr17/<name>
git rebase upstream/master                    # make sure it's current
git push origin pr17/<name>
gh pr create --repo srwi/keypeek --base master --head urielpappilon:pr17/<name>
```

### Add a new local change

```sh
git checkout -b mine/<short-name> upstream/master
# ...edit, commit...
git checkout main
git merge --no-ff mine/<short-name>           # or cherry-pick if you want linear history
git push origin main mine/<short-name>
```

## Build

ARM macOS:

```sh
export PATH="/opt/homebrew/opt/rustup/bin:$HOME/.cargo/bin:$PATH"
cargo bundle --release        # produces target/release/bundle/osx/KeyPeek.app
# To install:
cp -R target/release/bundle/osx/KeyPeek.app /Applications/
```

For two parallel installs (e.g. `main` and a comparison branch), patch the bundle ID on the second one before installing:

```sh
/usr/libexec/PlistBuddy -c "Set :CFBundleIdentifier dev.srwi.keypeek.<suffix>" \
    /path/to/Variant.app/Contents/Info.plist
/usr/libexec/PlistBuddy -c "Set :CFBundleName KeyPeek-<suffix>" \
    /path/to/Variant.app/Contents/Info.plist
codesign --force --deep --sign - /path/to/Variant.app
```

## Why this shape

- **`master` mirrors upstream → never pushed.** Keeps the fork's `master` from drifting. Any local commit on `master` is operator error.
- **Linear `main`.** Easy to read, easy to rebase, every commit traces to an upstream PR via its `cherry picked from` trailer.
- **Atomic topic branches where possible.** Each is a one-purpose PR-ready unit. Where commits genuinely depend on each other (the CLI/IPC stack, the render-pipeline changes), they stack rather than pretend to be independent — the stacking is documented in the table above.
- **Author preserved.** All cherry-picks use `-x` and keep the original PR author on `Author:`. The `cherry picked from commit <sha>` trailer points back to the upstream commit.
- **`rerere` enabled.** When upstream lands a change that conflicts with `main`, git remembers your resolution the first time so subsequent rebases auto-resolve.
