# How v0.2.0 was built

This document records how the binary asset in this release was produced.

The important point is:

> `cedar-render-foundation-dist` does not build Chromium / CEF itself.
> v0.2.0 was produced by the `cedar-render-foundation` source repository's
> own build scripts, then the resulting `minimal` distribution was published
> here.

## Source of truth

The build comes from the `cedar-render-foundation` repository, which stores the
pinned CEF / Chromium core source, the **internalized raw-snapshot managed
source**, plus its build entry points:

- `scripts/prepare-build.sh` — prepare pinned deps, tools, and metadata (online).
- `scripts/build.sh` — build the Linux x64 Release target and run the official `minimal` packaging.
- `scripts/package-release.py` — produce the externally named candidate + `SHA256SUMS` + `provenance.json`.
- `docs/build-baseline.md` — the pinned baseline and build-argument source.

Unlike v0.01, this release does **not** use the `cedar-c2v-cpp`
`cef-custom-build/` pipeline, and does **not** apply the raw-snapshot patch at
build time — the raw-snapshot capability lives as managed source in the
foundation repository and is compiled directly.

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
| foundation repo (library build source) | commit `1d6e5ca73a802b8041244eb6b6b1e460de258276` (clean, `origin=built-here`) |
| foundation repo (release anchor) | commit `63cb4328` (PR #8 squash-merged to `main`), tag `v0.2.0` |
| Distribution | Linux x64, Release, CEF `minimal`, non-component |
| Archive format | `tar.bz2` |

> The authoritative per-build identity is recorded in the `provenance.json`
> asset published with this release (`source.foundation_commit`,
> `build_identity.*`). The library bytes were built from `1d6e5ca7`; the
> release anchor commit `63cb4328` is the squash merge of PR #8 on `main`
> (the follow-up commit on the branch head was documentation only and does not
> change the library).

## Build arguments

Authoritative `args.gn` used for this build
(`chromium/src/out/Release_GN_x64/args.gn`; identical to v0.1.0 — the
raw-snapshot capability changes source only, not any GN build flag):

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

## Cedar customization: native raw snapshot

The single Cedar functional customization in this release is the internalized
raw snapshot:

| Item | Value |
| --- | --- |
| managed source (implementation) | `chromium/src/cef/libcef/browser/vg_raw_snapshot.cc` |
| managed source (public contract) | `chromium/src/cef/include/cedar/cedar_raw_snapshot.h` |
| managed source (unified entry) | `chromium/src/cef/include/cedar/cef_vg_api.h` (aggregates Cedar public C APIs; currently just the raw header) |
| BUILD.gn integration | `chromium/src/cef/BUILD.gn` `libcef_static` sources add `vg_raw_snapshot.cc`; both public headers ship via `cef_paths2.gypi` `includes_common` into `include/cedar/` |
| exported symbol | `cef_request_raw_snapshot` (covered by the existing `libcef.lst` `global: cef_*`) |
| origin | `cedar-c2v-cpp` v2.1.2 (commit `bc0bff22396f454ebf5fb2f16dbb79e5628e6365`), `cef-custom-build/patches/0001-b1-native-raw-snapshot.patch`, migrated line-for-line |

After internalization the compiled source is the single source of truth; no
hand-synced executable patch is maintained. The source is marked with
`// [CEDAR]: issue #7`. The build does **not** rewrite `cef_version.h` (the CEF
internal version stays `152.0.6+g708dc14+chromium-152.0.7977.83`).

## Build flow

```
pinned CEF / Chromium source + internalized raw managed source (foundation repo)
        │
        ├─ pinned depot_tools
        └─ PGO profiles
        │
        ▼
scripts/prepare-build.sh      (prepare pinned deps / tools / metadata)
        │
        ▼
scripts/build.sh --target cefsimple   (build libcef incl. vg_raw_snapshot)
        │
        ▼
scripts/package-release.py --version v0.2.0   (calls build.sh --package, then
        │                                       re-verifies artifact identity,
        │                                       renames, writes SHA256SUMS +
        │                                       provenance.json)
        ▼
scripts/verify/run-render-check.sh <pkg>   (OSR render regression)
scripts/verify/run-raw-check.sh <pkg>      (raw pixel acceptance; public-header
        │                                   compile + dlsym in-package libcef.so
        │                                   + dladdr load check + close-dropped)
        ▼
scripts/package-release.py --version v0.2.0 --runtime-verified <result.json>
        │                                   (backfill runtime verification)
        ▼
publish as v0.2.0 asset (this repository release)
```

### 1. Prepare a build host

Use a Linux x64 environment that can build Chromium / CEF. See
`docs/build-baseline.md` and `docs/build-workflow.md` in the
`cedar-render-foundation` repository for the host requirements and the actual
prepare order.

Reference acceptance environment: Ubuntu 24.04, 8 vCPU, ~16 GiB RAM. The build
requires enough disk and memory for a full Chromium / CEF source build.

### 2. Produced CEF archive

The build produces, under the foundation repository:

```
chromium/src/out/Release_GN_x64/cef_distrib/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

The archive keeps the standard CEF `minimal` distribution layout, with the
Cedar public headers added under `include/cedar/`.

### 3. Verify functional rendering and raw snapshot

This release was verified with the foundation repository's verify scripts:

```
# run-render-check.sh
painted=1  red=1 green=1 blue=1  timed_out=0  exit code 0

# run-raw-check.sh (public header compile as C11 + C++20, dlsym in-package libcef.so)
reject(bad-id / null-cb) + solid red/green/blue/black/white
+ stride==w*4 + root-surface alpha=255 + alpha-opaque-composite
+ exactly one callback per request + close-dropped (DROPPED settled once)
dladdr loaded libcef.so SHA-256 == in-package libcef.so SHA-256
exit code 0
```

Unlike v0.1.0, a symbol check for `cef_request_raw_snapshot` **is** an
acceptance criterion for this release: the export must be present and must
produce correct pixels.

### 4. Record the published artifact identity

| Item | Value |
| --- | --- |
| original archive | `cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2` |
| size | `321,472,645` bytes |
| archive SHA-256 | `ce9b6af5fafcc5bca19bcd9ece92b86a031139947ab6c528815f51057867a3a1` |
| `libcef.so` SHA-256 | `122fbf4b1e5cacf2251c0afc4cc406712027b78e233bc3e2a032d29d5f29171b` |
| custom export | `cef_request_raw_snapshot` |

These values are also recorded machine-readably in the `provenance.json` asset
(schema 2) published with this release. `make_distrib` did not strip the
library, so the in-package `libcef.so` and the built `libcef.so` share the same
SHA-256 (`archive_sha256` / `built_libcef_sha256` cross-check).

### 5. Rename for the Cedar release

`scripts/package-release.py` does this automatically (it can also be done by
hand); the archive is **not** extracted and repacked, only the external
filename changes:

```bash
cp \
  chromium/src/out/Release_GN_x64/cef_distrib/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2 \
  cedar-render-foundation-v0.2.0-chromium-152.0.7977.83-linux-x64.tar.bz2

sha256sum cedar-render-foundation-v0.2.0-chromium-152.0.7977.83-linux-x64.tar.bz2
```

The renamed asset keeps the same size and SHA-256 as the archive it was copied
from.

## Relationship to v0.01 and v0.1.0

v0.2.0 is a **separate** foundation release:

- v0.01 is the `cedar-c2v-cpp`-pipeline build **with** the raw-snapshot export.
- v0.1.0 is a `cedar-render-foundation`-repo build **without** any custom export.
- v0.2.0 is a `cedar-render-foundation`-repo build **with** the raw-snapshot
  export, now compiled from internalized managed source rather than an applied
  patch.

Their byte contents and SHA-256 values differ. Each release's artifact identity
stands on its own, and consumers must pin the specific release they depend on.
A consumer that needs the raw-snapshot capability should prefer v0.2.0 over the
older patch-based v0.01.

## Rebuilding from the same inputs

For the closest reproduction of this release:

1. Check out `cedar-render-foundation` at tag `v0.2.0` (or the library build
   source commit `1d6e5ca7`).
2. Ensure the host can build Chromium / CEF (see `docs/build-baseline.md`).
3. Run `scripts/prepare-build.sh`, then
   `scripts/package-release.py --version v0.2.0`.
4. Run `scripts/verify/run-render-check.sh` and
   `scripts/verify/run-raw-check.sh` and confirm both pass, including the
   `cef_request_raw_snapshot` export.
5. Backfill runtime verification with
   `scripts/package-release.py --version v0.2.0 --runtime-verified <result.json>`.

The checksums above identify the originally published binary asset; a fresh
Chromium / CEF rebuild is not guaranteed to be byte-identical. The presence and
behavior of the raw-snapshot export are the stable acceptance criteria.
