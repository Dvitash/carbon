# Spark fork of Carbon

This is Dvitash's fork of [Chrrxs/carbon](https://github.com/Chrrxs/carbon). Spark (DGX Spark, ARM64 Linux,
Roblox Studio under Wine/Proton) builds Carbon from source because upstream publishes x86_64 Linux builds only.
This fork carries the Spark patches on top of an upstream release tag.

## Branches

- `spark`: an upstream release tag plus the patches below. Spark builds and runs this branch.
- `main` is an old copy of upstream `main`; the other branches are earlier pull-request branches (for example
  `wine-studio-host`, merged upstream as #32). Nothing on Spark builds them.

The version stays the upstream tag's (`26.10.281809` for tag `v26.10.281809`): `scripts/build-studio-plugin` and
`scripts/build-version` accept only upstream's `YY.M.DHHMM` format, and the CLI and its embedded plugin are built
with the same version. So `carbon --version` shows upstream's version, and `carbon-update` records the fork commit
next to the binary (see "Build and install").

## Patches

1. `Build a Compatibility-migrated Lighting as Voxel` (`cli/src/source_wire.rs`, `BUILD_CACHE_SCHEMA` 4 → 5 in
   `cli/src/project.rs`). Roblox retired Compatibility lighting: Studio migrates it to Voxel on every load,
   marks the place with the `RBX_LightingCompatibilityMigrated` attribute and shows the "Lighting Technology
   Migration" dialog. Building such a place back as Compatibility made Studio migrate and prompt again, and the
   dialog swallowed synthetic Ctrl+S on Spark. A migrated place now builds as Voxel. Tests:
   `source_wire::tests` (`restores_technology_studio_changed_on_load`,
   `builds_voxel_after_compatibility_sunset_migration`, `restores_compatibility_without_completed_migration`).
2. `FORK.md` (this file).

## Fixed upstream; why the fork moved past v26.10.101210

Spark ran `v26.10.101210` until 2026-10-29. On it, the first source edit after every fresh `carbon serve` of a
project with a properties-only engine anchor (an anchor with `$properties` and no `$path` or mapped children:
DSV99's `Lighting`, Hotwire's `HttpService` and `Workspace`) wedged source sync. The serve looped
`Could not commit project source change: addition <id> already exists` and Studio stopped receiving edits.
The id is `stable_ref("anchor:<Class>:<Name>")`, so it was the same on every serve: `c24f91e4…` is
`anchor:Lighting:Lighting`, `e6fe3ddb…` is `anchor:HttpService:HttpService`.

Cause: the serve projection held only mapped subtrees and the ancestors of mapped roots. A properties-only
anchor is neither, so it was missing from the projected snapshot. Each source re-evaluation created the anchor
again, the diff reported it as an addition, and `apply_changes` in `cli/src/artifact_store.rs` refused it
because the artifact tree already had that id. Upstream `1c611e4` ("Project properties-only engine anchors during
serve", in `v26.10.281809`) records every anchor route and projects each anchor that is not filesystem-owned as
a routing ref. Its regression test, `project::tests::projected_watcher_edit_keeps_properties_only_engine_anchor_in_projection`,
fails on `ee5a84a` (the commit before the fix) with "properties-only anchor is missing from the projection" and
passes from `1c611e4` on. Keep that test when rebasing.

## Build and install

`~/.local/bin/carbon-update` (source: `~/Projects/tern-worktree/spark/carbon-update`) builds `spark` in
`~/.cache/carbon-src` and installs it to `~/.local/lib/carbon/carbon`. It takes the version from the newest
upstream tag `spark` contains, writes `<version>-spark+<commit>` to `~/.local/lib/carbon/carbon.source`, and
rebuilds only when either changed. It warns when upstream has a newer release than the one `spark` is on.

Tests read Studio's reflection, so run them with the Spark Studio environment:

```sh
ROBLOX_STUDIO_WINE_LAUNCHER=$HOME/.local/bin/spark-studio-wine-exec \
ROBLOX_STUDIO_EXE=$HOME/.local/share/spark-studio/current/RobloxStudioBeta.exe \
	cargo test --locked -p carbon --lib
```

Running serves keep the binary they started with; a serve picks up a new install when it restarts.

## Rebase onto a new upstream tag

```sh
cd ~/src/carbon                          # remotes: origin = Chrrxs/carbon, fork = Dvitash/carbon
git fetch origin --tags && git fetch fork
git switch spark
git rebase --onto vNEW $(git describe --tags --abbrev=0 --match 'v*' spark) spark
# Drop a patch if upstream now contains it. Keep the tests when resolving conflicts.
ROBLOX_STUDIO_WINE_LAUNCHER=... ROBLOX_STUDIO_EXE=... cargo test --locked -p carbon --lib
git push --force-with-lease fork spark
carbon-update                            # builds and installs the new spark head
```

Then restart one worktree's serve and edit a mapped `src/` file: the change must reach Studio's `.Source`
(check with `execute_luau`) and the serve pane must stay free of `already exists` errors.
