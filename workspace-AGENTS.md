# Canonical openKylin XR workspace

- Each component repository here is its only openKylin implementation/packaging maintenance entry. The machine-readable registry is `xr-image-tools/repositories.json`; source selection is `xr-image-tools/components.lock.json`.
- Build into a new external output directory with `xr-build-components`. Do not compile code from historical qiyu component checkouts, stale prefixes, or an arbitrary worktree selected by cwd. Git object caches and immutable upstream/vendor inputs are allowed and must match their locks.
- Kernel implementation remains in `bitsk/qiyu-kernel`; `xr-kernel` is the canonical source-selection/package wrapper. Do not recreate a second driver patch stack.
- Versioned cross-repository guidance is in `openkylin-xr-docs/wiki/component-migration.md`. Keep API headers, runtime code and package recipes in their owning repositories.
- Preserve pre-existing candidate edits, author metadata, published tags and release baselines. A package build or source migration does not establish fresh physical acceptance. Keep vendor weights and per-device calibration outside public Git.

## Publish completed changes to GitHub main

- Before closing a task, commit the completed changes and publish them to `main` in each owning GitHub repository after the applicable checks pass. Follow the repository's pull request process when branch protection requires it, and confirm the merge.
- For cross-repository changes, verify producers and consumers together. Publish the component commits first, then publish the affected recipes and `xr-image-tools/components.lock.json` with the exact commits reachable from GitHub `main` and the updated public-header digests.
- Check the affected `refs/heads/main` on GitHub before reporting completion. Confirm that each intended change is present, including changes merged as equivalent patches. Preserve remote history. Report the affected repository, commit and concrete blocker if publication cannot finish.
