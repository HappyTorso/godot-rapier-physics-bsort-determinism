# godot-rapier-physics — B-sort determinism patch

This is a **patched fork of [godot-rapier-physics](https://github.com/appsinacup/godot-rapier-physics) v0.8.36**
(the `single-enhanced-determinism` 2D flavor), with one small, targeted change on
top of the upstream release. It is not a general-purpose replacement for the
upstream addon — see "When you'd want this" below before using it.

Original project README (unmodified, upstream authorship/license notice):
[`UPSTREAM_README.md`](UPSTREAM_README.md).

## The change: sort broad-phase query results by stable collider identity ("B-sort")

**File:** `src/spaces/rapier_space_body_helper.rs`, fn `rapier_intersect_aabb`.
**Patch:** [`patches/b-sort-contact-order-determinism.patch`](patches/b-sort-contact-order-determinism.patch)

`CharacterBody2D.move_and_slide()`'s kinematic motion recovery and cast
tie-break consume the broad-phase query's result order. That order comes from
Rapier's BVH traversal, which — after `import_state()` (i.e. after a snapshot
restore, as used by rollback netcode / deterministic replay) — is **not
guaranteed to match the order it had during live simulation**, even though all
collider positions restore byte-identically ([godot-rapier issue #534](https://github.com/appsinacup/godot-rapier-physics/issues/534)).

When a kinematic body overlaps two or more colliders at a near-tie distance
(a body wedged in a corner, for example), a shuffled result order can flip
*which* collider the recovery/cast logic resolves against — so a live
recording and a state-restore-then-resimulate replay of the exact same frame
can diverge.

The patch sorts the broad-phase result set by each collider's stable Godot
identity (`UserData{part1, part2}`, assigned at collider creation, independent
of Rapier's internal handle/traversal order) before it reaches the
recovery/cast callers. This makes the consumed order identical whether the
body arrived at this frame via live simulation or via `import_state()` +
resimulate.

**This is a pure reordering of an already-computed result set — no collision
math, contact resolution, or physics behavior is changed.** Two bodies at a
genuine tie will now consistently pick the same one of the two, rather than
picking whichever the (order-unstable) traversal happened to enumerate first.

## When you'd want this

You almost certainly do **not** need this patch for normal single-player or
non-rollback multiplayer use — stock godot-rapier is more tested, more
optimized, and gets upstream bugfixes you'd otherwise have to manually port.

This patch matters specifically if your project:
- Captures/restores physics state via Rapier's `import_state()` /
  `export_state()` (snapshot-based save states, deterministic replay
  recording, or **rollback netcode**), *and*
- Needs record → save-state → restore → resimulate to reproduce the **exact
  same result**, frame for frame (not just "close enough" / not just
  bit-identical *saved bytes*, but bit-identical *behavior after restore*),
  *and*
- Uses `CharacterBody2D` kinematic bodies that can end up overlapping multiple
  colliders at once (crowded scenes, corner-wedging, tightly packed gameplay
  entities).

If your project doesn't restore mid-match physics state and resimulate from
it, this patch changes nothing you'd ever notice — skip it and use stock
godot-rapier.

## Known limitation

This fixes the *within-body* class of the contact-selection flip (one body
choosing between two-or-more collision candidates). A related but distinct
*cross-body* class (two separate bodies mutually resolving against each
other, order-dependent) is **not** fixed by this patch alone — see godot-rapier
issue #534 for the broader tracking context. If you hit an ordering-dependent
divergence this patch doesn't resolve, that's the class it's outside of.

## Platform status

| Platform | Status |
|---|---|
| macOS arm64 | Patched and validated |
| Windows x86_64 | Built via [`.github/workflows/build-windows-bsort.yml`](.github/workflows/build-windows-bsort.yml) — run it manually (Actions tab → "Build patched Windows dll (B-sort)" → Run workflow) and download the `libgodot_rapier-windows-x86_64-msvc-bsort` artifact |
| Linux, other Windows arches | Not yet built — extend the workflow with additional `arch` values / a Linux job following the same `./.github/actions/build` composite action if you need them |

## Building

The build uses upstream's existing `./.github/actions/build` composite action
(same Rust toolchain pin, target setup, and feature flags upstream CI uses),
just scoped to one config. To build locally instead of via Actions, see
`rust-toolchain.toml` for the pinned nightly and use:

```sh
cargo +nightly-2025-12-12 build --target=<your-target-triple> --profile=release \
  --features="enhanced-determinism,serde-serialize,experimental-threads,register-docs,single-dim2,api-custom" \
  --no-default-features
```

## Using the built binary

Drop the built library into your own project's `addons/godot-rapier2d/bin/`,
matching the filename `godot-rapier2d.gdextension` already expects for that
platform (e.g. `libgodot_rapier.windows.x86_64-pc-windows-msvc.dll`). No
`.gdextension` changes are needed — it's a drop-in replacement for the stock
binary at the same path.

## License

MIT, inherited from upstream — see [`LICENSE`](LICENSE) and
[`UPSTREAM_README.md`](UPSTREAM_README.md) for original authorship
(Fabrice Cipolla, Sp3ctralCat, Dragos Daian and contributors).
