# How v0.1.0 was built

This document records how the binary asset in this release was produced.

The important point is:

> `cedar-render-foundation-dist` does not build Chromium / CEF itself.
> v0.1.0 was produced by the `cedar-render-foundation` source repository's
> own build scripts, then the resulting `minimal` distribution was published
> here.

## Source of truth

The build comes from the `cedar-render-foundation` repository, which stores the
pinned CEF / Chromium core source plus its build entry points:

- `scripts/prepare-build.sh` — prepare pinned deps, tools, and metadata (online).
- `scripts/build.sh` — build the Linux x64 Release target and run the official `minimal` packaging.
- `docs/build-baseline.md` — the pinned baseline and build-argument source.

Unlike v0.01, this release does **not** use the `cedar-c2v-cpp`
`cef-custom-build/` pipeline, and does **not** apply the raw-snapshot patch.

## Pinned inputs

| Item | Value |
| --- | --- |
| CEF version | `152.0.6+g708dc14+chromium-152.0.7977.83` |
| CEF branch | `7977` |
| CEF commit | `708dc140cbc3286826a8abef89dc23a44ff9ea72` |
| Chromium | `152.0.7977.83` |
| Chromium checkout | `refs/tags/152.0.7977.83` |
| Chromium commit | `79460ebecaa5625e57a5fb679a735659e73dc687` |
| depot_tools commit | `81577f19a8497ba7e41afac322e8f03553a863ec` |
| foundation repo commit | `63208c7e` (clean) |
| Distribution | Linux x64, Release, CEF `minimal`, non-component |
| Archive format | `tar.bz2` |

## Build arguments

Authoritative `args.gn` used for this build
(`chromium/src/out/Release_GN_x64/args.gn`, copied to `.build/args.gn.used`):

```
blink_heap_inside_shared_library=true
clang_use_chrome_plugins=false
disable_fieldtrial_testing_config=true
enable_background_mode=false
enable_backup_ref_ptr_support=false
enable_downgrade_processing=false
enable_linux_installer=false
enable_precompiled_headers=false
enable_resource_allowlist_generation=false
enable_widevine=true
forbid_non_component_debug_builds=false
is_cfi=false
is_component_build=false
is_debug=false
is_official_build=true
optimize_webui=true
symbol_level=1
target_cpu="x64"
use_partition_alloc_as_malloc=false
use_qt5=false
use_qt6=false
use_remoteexec=false
use_siso=false
use_sysroot=true
```

The build uses local Ninja (`use_remoteexec=false`, `use_siso=false`), PGO
profiles, and the official CEF driver pinned to the selected CEF revision.

## Cedar customization

None for this release.

v0.1.0 is a clean build of the pinned CEF / Chromium baseline. It does **not**
apply the `0001-b1-native-raw-snapshot.patch` used by v0.01, so the built
`libcef.so` does **not** export `cef_request_raw_snapshot`.

## Build flow

```
pinned CEF / Chromium source (cedar-render-foundation repo)
        │
        ├─ pinned depot_tools
        └─ PGO profiles
        │
        ▼
scripts/prepare-build.sh   (prepare pinned deps / tools / metadata)
        │
        ▼
scripts/build.sh --target cefsimple   (build libcef + chrome_sandbox)
        │
        ▼
scripts/build.sh --package   (package CEF minimal distribution)
        │
        ▼
scripts/verify/run-render-check.sh   (render_check functional verification)
        │
        ▼
record artifact identity (size + SHA-256)
        │
        ▼
rename archive for Cedar release
        │
        ▼
publish as v0.1.0 asset
```

### 1. Prepare a build host

Use a Linux x64 environment that can build Chromium / CEF. See
`docs/build-baseline.md` and `docs/build-workflow.md` in the
`cedar-render-foundation` repository for the host requirements and the actual
prepare order.

Reference acceptance environment: Ubuntu 24.04.4, 8 vCPU, ~16 GiB RAM. The
build requires enough disk and memory for a full Chromium / CEF source build.

### 2. Produced CEF archive

The build produces, under the foundation repository:

```
chromium/src/out/Release_GN_x64/cef_distrib/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

The archive keeps the standard CEF `minimal` distribution layout.

### 3. Verify functional rendering

This release was verified with the foundation repository's
`scripts/verify/run-render-check.sh`, which extracts the `minimal` package,
builds a minimal CEF program against it, and renders a local HTML page using
off-screen rendering with software compositing (no X server required).

Verified result:

```
painted=1  red=1 green=1 blue=1  timed_out=0  exit code 0
```

Note: v0.1.0 does not export `cef_request_raw_snapshot`, so a symbol check for
that export is **not** an acceptance criterion for this release (and must not be
present). The acceptance for this release is that the standard CEF `minimal`
package builds and renders HTML correctly.

### 4. Record the published artifact identity

| Item | Value |
| --- | --- |
| original archive | `cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2` |
| size | `321,479,861` bytes |
| archive SHA-256 | `12e52fe818053a4e837a0c87b410cb22d4018b62096bb020439f71618b6a261a` |
| `libcef.so` SHA-256 | `8704a8783110e906a8ece432541064036f29c090ac04eb4485440c886b5f5579` |
| custom export | none |

### 5. Rename for the Cedar release

Publish the generated archive under the Cedar-facing name, without extracting
and repacking it (only the external filename changes):

```bash
cp \
  chromium/src/out/Release_GN_x64/cef_distrib/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2 \
  cedar-render-foundation-v0.1.0-chromium-152.0.7977.83-linux-x64.tar.bz2

sha256sum cedar-render-foundation-v0.1.0-chromium-152.0.7977.83-linux-x64.tar.bz2
```

The renamed asset keeps the same size and SHA-256 as the archive it was copied
from.

## Relationship to v0.01

v0.1.0 is a **separate** foundation release, not a replacement for v0.01:

- v0.01 is the `cedar-c2v-cpp`-pipeline build **with** the raw-snapshot export.
- v0.1.0 is a `cedar-render-foundation`-repo build **without** any custom export.

Their byte contents and SHA-256 values differ. Each release's artifact identity
stands on its own, and consumers must pin the specific release they depend on.

## Rebuilding from the same inputs

For the closest reproduction of this release:

1. Check out `cedar-render-foundation` at commit `63208c7e`.
2. Ensure the host can build Chromium / CEF (see `docs/build-baseline.md`).
3. Run `scripts/prepare-build.sh`, then `scripts/build.sh --package`.
4. Run `scripts/verify/run-render-check.sh` and confirm it renders HTML correctly.
5. Copy the generated tarball to the Cedar-facing filename without repacking it.

The checksums above identify the originally published binary asset; a fresh
Chromium / CEF rebuild is not guaranteed to be byte-identical.
