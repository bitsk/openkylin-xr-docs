# Canonical openKylin XR workspace

- Each component repository here is its only openKylin implementation/packaging maintenance entry. The machine-readable registry is `xr-image-tools/repositories.json`; source selection is `xr-image-tools/components.lock.json`.
- Build into a new external output directory with `xr-build-components`. Do not compile code from historical qiyu component checkouts, stale prefixes, or an arbitrary worktree selected by cwd. Git object caches and immutable upstream/vendor inputs are allowed and must match their locks.
- Kernel implementation remains in `bitsk/qiyu-kernel`; `xr-kernel` is the canonical source-selection/package wrapper. Do not recreate a second driver patch stack.
- Versioned cross-repository guidance is in `openkylin-xr-docs/wiki/component-migration.md`. Keep API headers, runtime code and package recipes in their owning repositories.
- Preserve pre-existing candidate edits, author metadata, published tags and release baselines. A package build or source migration does not establish fresh physical acceptance. Keep vendor weights and per-device calibration outside public Git.
