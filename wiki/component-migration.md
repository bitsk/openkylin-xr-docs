# Component migration and maintenance

The canonical openKylin workspace contains one independently maintained repository per component. Runtime builds use only commits selected by `xr-image-tools/components.lock.json`; historical qiyu component checkouts are source/evidence archives.

The main kernel implementation remains in [bitsk/qiyu-kernel](https://github.com/bitsk/qiyu-kernel). The `xr-kernel` repository owns source selection and package construction, not another driver patch stack.

| Capability | Maintained owner | Migration and acceptance boundary |
| --- | --- | --- |
| Shared camera producer | xr-device-io | OV7251/QCFT and shared hand tap retained; IMX capture, RAW snapshots and calibration exporter restored. Original camera ownership and timing gates retained |
| Native device identity | xr-device-io / xr-controller-tracking / xr-debug-console | Restore actual board identity checks and openKylin radio support. Display naming is independent of hardware/ABI identifiers |
| Head wire and motion audit | xr-head-tracking | Production v2 headers retained; wire contract and read-only motion audit added. No replacement timestamps or fabricated poses |
| Bounded SDK logs and FastRPC headers | xr-controller-tracking / xr-kernel | Recover the SDK log shim source from the archived build, package it with the ARM32 worker and consume the locked kernel FastRPC UAPI. Capsule checksums replace a historical binary hash |
| Controller namespace and SDK | xr-session-manager / xr-controller-tracking | OpenKylin root marker and trusted system opt alias retained in packaged launcher. One camera owner; original SDK bytes imported separately |
| Head tracking recovery | xr-monado | Existing inertial/reacquisition implementation matches the old accepted header except owned SDK includes. Position inference does not become optical tracking |
| Original controller models | openkylin-xr-integration | Native model package recipe migrated; grip/aim remain separate from model frame. Original model payload imported by digest |
| Camera preview phases | xr-device-io | ACP2 joined phase metadata restored; shared preview is nonblocking. Restoration is not a new physical phase acceptance claim |
| WebRTC distribution dependencies | xr-debug-console | Five byte-preserved official openKylin plugins and copyright are pinned into an app-scoped package. Avoid the unrelated unavailable dependencies of the complete bad plugin set; system GStreamer paths remain unchanged |
| Debug video recovery | xr-debug-console | Existing backend NACK/RTX controls retained; GPU native keyframe input restored. Installed GPU helpers are recognized as encoded streams; video assets, Mesa ICD and timing helpers use their owning package paths. No timer-driven periodic keyframes enabled |
| Passthrough camera labels | xr-debug-console | Existing measured camera role mapping retained. No heuristic left/right renaming |
| Boot selector ordering | xr-image-tools | Explicit one-shot Debian choice takes priority over persistent defaults; old and new config markers accepted. No partition or running baseline changes |
| USB console lifecycle | openkylin-xr-integration | No TTY reset/hangup configuration restored. Avoid blocking PID1 on an unconnected ACM port |
| ALSA hardware alias | openkylin-xr-integration | Actual Qiyu MIX card filename restored. Hardware card matching names remain unchanged |
| Thermal configuration | openkylin-xr-integration | Runtime equivalence checked; installer now accepts installed openKylin. Factory policy retained |
| Python GPIO ABI | openkylin-xr-integration | Pinned native CPython3.12 gpiod2.2.4 build recipe migrated. Explicit native dependency package; not a host wheel substitution |
| Android Linux launchers | openkylin-xr-integration | Source and controlled selector recipe migrated as an optional device utility. Not installed or activated by this source migration |
| Scheduling and fixed PD kernel | xr-kernel build wrapper | Pin the verified canonical kernel commit with IFE-Lite, square crop, fixed5V PD and optional scheduling configs. Driver/config implementation stays only in bitsk/qiyu-kernel |
| Graphics and runtime compatibility | xr-monado / xr-mesa | Accepted display split remains in default Mesa patch; Gamescope/native interop adaptations migrated. Negative performance candidates remain outside the default series |
| Session code installation | xr-session-manager | Systemd and child processes execute installed module paths; profile/capsule preparation is explicit. Private capsules carry data/libraries and package-derived native artifacts |
| Desktop source ownership | xr-shell / xr-spatial-kit / xr-system-ui / xr-dock / xr-wayland / xr-environments | Components retain their single owners and are selected through one source lock. Pre-existing unvalidated worktree edits are preserved separately |
| OpenXR loader dependency | openkylin-xr-integration | Exact upstream ARM64 package import is locked because the openKylin archive lacks the loader. The bundle carries it explicitly and pins transitive runtime package versions |
| OEM assets | xr-vendor-assets | Single checksummed firmware/SDK cache and generated Debian packages retained. No per-device calibration or model weights published |
| Quest hand backend | xr-hand-tracking | Original-weight backend and standard joint mappings retained. Internal V66 logs identified VTCM acquisition failure; a 16 KiB outer-graph hint passed identical-output replay and 1759 concurrent live computations. Positive live-hand accuracy and physical tap acceptance remain pending |

## Building and installing

`xr-build-components build --profile vr-hands --output NEW_DIRECTORY` checks out the exact GitHub commits, builds the shared SDKs before consumers, and emits Debian packages plus a SHA-256 manifest and local APT index. `--source-cache` supplies only Git objects; uncommitted files never enter this build. Toolchains, distribution sysroots and native third-party dependency packages are explicit inputs. Resume rejects changed locks or toolchain configuration.

`xr-image-builder assemble --system openkylin --profile vr-hands --base-archive BASE --package-manifest BUILD/build-manifest.json --output BUNDLE` verifies package/source identity and file ownership, then exports the immutable base, board overlay and package repository. The generated installer pins exact versions. Assembly does not run maintainer scripts, activate XR, or establish physical device acceptance.

After package installation, `xr-session configure --profile EXISTING_PRIVATE_PROFILE --output NEW_PROFILE --capsule NEW_PRIVATE_CAPSULE` prepares a separate asset capsule and switches program references to the installed package entrypoints. It does not change boot selection or running services. Calibration, upstream SteamVR, original model weights and the externally licensed pose observer remain explicit private/upstream inputs.

A complete clean image and hardware acceptance require applying packages/dependencies and device configuration. They are separate from successful package construction. Native package activation also requires the continuous head observer, DSP entropy mapping, installed exposure-table path, immutable installed controller plan, and stable private Mesa ICD filename. These paths are now exercised by the installed session.

The hand backend uses the original DSP library and weights. A controlled concurrent run completed 1759 detector/backbone/fusion/host-decoder groups after reducing the outer VTCM hint, with no execution error. Both saved-hand outputs and an exact failed detector input matched the default configuration byte for byte. This is an execution/resource result; physical gestures and double-tap thresholds still require wearer validation. Root session controls subscribe to the existing IMU relay, and tracking reacquisition requires gesture release before accepting a new action.
