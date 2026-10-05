# Cedar Render Foundation v0.2.0

A Cedar Render Foundation distribution built directly from the
`cedar-render-foundation` source repository.

Its purpose is the same as earlier foundation releases:

> Provide upper-layer projects such as `cedar-c2v-cpp` with a compiled and
> verified Chromium / CEF foundation package, so Chromium / CEF does not need
> to be rebuilt for every normal build.

v0.2.0 additionally **internalizes the native raw-snapshot capability as
managed source** in the foundation repository, so the resulting `libcef.so`
exports `cef_request_raw_snapshot` without the consumer carrying a
`cef-custom-build/` patch pipeline.

## Package

```
cedar-render-foundation-v0.2.0-chromium-152.0.7977.83-linux-x64.tar.bz2
```

## Base

This release is based on:

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

| Item | Value |
| --- | --- |
| Cedar version | `v0.2.0` |
| CEF | `152.0.6+g708dc14` |
| Chromium | `152.0.7977.83` |
| Platform | Linux x64 |
| Distribution | CEF `minimal` |
| Custom capability | **native raw snapshot** (`cef_request_raw_snapshot`) |

> The Cedar version `v0.2.0` is a **foundation release anchor**. The internal
> CEF / Chromium version in `cef_version.h`
> (`152.0.6+g708dc14+chromium-152.0.7977.83`) is **not** rewritten; v0.2.0 does
> not change the CEF version.

## How this release was built

See [`BUILD.md`](BUILD.md) for the build provenance and reproduction flow.

Like v0.1.0 — and unlike v0.01, which was produced by the `cedar-c2v-cpp`
`cef-custom-build/` pipeline — v0.2.0 was built directly from the
`cedar-render-foundation` source repository, which keeps the pinned CEF /
Chromium core source and its own build scripts (`scripts/prepare-build.sh`,
`scripts/build.sh`, `scripts/package-release.py`).

The difference from v0.1.0 is the raw-snapshot capability: v0.1.0 is a clean
baseline with no custom export, while v0.2.0 compiles the internalized
raw-snapshot source into `libcef.so`.

## What's included

v0.2.0 keeps the standard CEF `minimal` distribution layout, plus the Cedar
public header directory `include/cedar/`.

The top-level directory inside the tarball remains:

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
```

Main contents:

```
cef_binary_..._linux64_minimal/
├── CMakeLists.txt
├── include/
│   └── cedar/
│       ├── cedar_raw_snapshot.h   (raw-snapshot public contract)
│       └── cef_vg_api.h           (unified Cedar public C API entry)
├── cmake/
├── libcef_dll/
├── Release/
│   └── libcef.so                  (exports cef_request_raw_snapshot)
└── Resources/
```

Key parts:

- `Release/libcef.so`: the main Chromium / CEF runtime library. For v0.2.0 it
  **also exports** `cef_request_raw_snapshot`.
- `include/cedar/`: Cedar public C API headers. `cedar_raw_snapshot.h` is the
  single source of truth for the raw-snapshot contract; `cef_vg_api.h` is the
  unified entry that aggregates Cedar public C APIs (currently just the raw
  header). Consumers should include `cef_vg_api.h`.
- `Resources/`: Chromium / CEF runtime resources.
- `include/`: standard CEF C/C++ headers.
- `libcef_dll/`: source for `libcef_dll_wrapper`.
- `cmake/` and `CMakeLists.txt`: CMake integration files.

This package is a complete CEF minimal distribution that can be consumed
directly by an existing CEF CMake project.

### What's not included

This package does not include:

- `cedar-c2v-cpp` application code
- the Node.js HTML-to-video service
- framemerger
- AV1 / WebM encoder code
- Cedar upper-layer worker / scheduling logic

Those remain part of `cedar-c2v-cpp`.

## Difference from official CEF

v0.2.0 is a Cedar-built CEF `minimal` distribution of the pinned
`152.0.7977.83` baseline that **adds exactly one** native export beyond
standard CEF:

```
cef_request_raw_snapshot
```

The implementation lives as managed source in the foundation repository
(`chromium/src/cef/libcef/browser/vg_raw_snapshot.cc`), internalized line-for-
line from `cedar-c2v-cpp` v2.1.2's `0001-b1-native-raw-snapshot.patch`. No
other symbol differs from a standard CEF `152.0.7977.83` minimal build.

## Using it in cedar-c2v-cpp

The distribution follows the standard CEF `minimal` layout, so an existing CEF
CMake integration can consume it the same way:

```
cedar-render-foundation-dist v0.2.0
        │
        │ download + SHA-256 verification
        ▼
cef/third_party/cef/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
        │
        ▼
CEF_ROOT
        │
        ▼
build cefservice / framemerger
        │
        ▼
runtime uses this libcef.so (with raw-snapshot export)
```

> **Compatibility note.** Unlike v0.1.0, v0.2.0 **does** export
> `cef_request_raw_snapshot`, so it satisfies the `cedar-c2v-cpp` `v2.1.2`
> capability gate that requires the raw-snapshot symbol. A consumer that
> previously used v0.01 for the raw-snapshot capability can move to v0.2.0, but
> the artifact identity differs (see below) and the consumer must re-pin the
> URL + SHA-256 and refresh its `CEF_ROOT`. The raw-snapshot contract is the
> public header `include/cedar/cedar_raw_snapshot.h` (reachable via
> `include/cedar/cef_vg_api.h`); consumers should include the shipped header
> instead of carrying their own copy.

## Artifact identity

Verified artifact:

| Item | Value |
| --- | --- |
| package size | `321,472,645` bytes |
| package SHA-256 | `ce9b6af5fafcc5bca19bcd9ece92b86a031139947ab6c528815f51057867a3a1` |
| `libcef.so` SHA-256 | `122fbf4b1e5cacf2251c0afc4cc406712027b78e233bc3e2a032d29d5f29171b` |
| custom export | `cef_request_raw_snapshot` |

This artifact identity is distinct from v0.01 and v0.1.0. v0.2.0 is a separate
build with different byte contents, so its SHA-256 values do not match the
other releases and must not be substituted for them.

Verify the download against the `SHA256SUMS` asset published alongside the
package. If the archive is repacked, modified, or rebuilt in the future, the
artifact identity must be recalculated and recorded even if the release version
stays the same. The byte-level SHA-256 identifies one specific build; the
**presence of the raw-snapshot export and its pixel behavior** are the stable
acceptance criteria.

## Functional verification

The published package was verified with the foundation repository's
out-of-tree consumers:

- `run-render-check.sh` — extracts the `minimal` package, builds a minimal CEF
  program against it, and renders a local HTML page using off-screen rendering
  with software compositing (no X server required):

  ```
  painted=1  red=1 green=1 blue=1  timed_out=0  exit code 0
  ```

- `run-raw-check.sh` — compiles the public entry `include/cedar/cef_vg_api.h`
  from the package's own `include/` as both C (`-std=c11`) and C++
  (`-std=c++20`), then `dlsym`s `cef_request_raw_snapshot` from the package's
  `libcef.so` and drives a real JS→native handshake to read pixels:

  ```
  reject(bad-id / null-cb) + solid red/green/blue/black/white
  + stride==w*4 + root-surface alpha=255 + alpha-opaque-composite
  + exactly one callback per request + close-dropped (DROPPED settled once)
  exit code 0
  ```

  A `dladdr` check confirms the actually loaded `libcef.so` matches the
  in-package library by SHA-256.

## Consumer

At the product level, consumers depend on:

```
Cedar version + Chromium baseline
```

For this release:

```
v0.2.0 + Chromium 152.0.7977.83  (with cef_request_raw_snapshot)
```
