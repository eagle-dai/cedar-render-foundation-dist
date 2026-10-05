# Cedar Render Foundation v0.1.0

A Cedar Render Foundation distribution built directly from the
`cedar-render-foundation` source repository.

Its purpose is the same as earlier foundation releases:

> Provide upper-layer projects such as `cedar-c2v-cpp` with a compiled and
> verified Chromium / CEF foundation package, so Chromium / CEF does not need
> to be rebuilt for every normal build.

## Package

```
cedar-render-foundation-v0.1.0-chromium-152.0.7977.83-linux-x64.tar.bz2
```

## Base

This release is based on:

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

| Item | Value |
| --- | --- |
| Cedar version | `v0.1.0` |
| CEF | `152.0.6+g708dc14` |
| Chromium | `152.0.7977.83` |
| Platform | Linux x64 |
| Distribution | CEF `minimal` |

## How this release was built

See [`BUILD.md`](BUILD.md) for the build provenance and reproduction flow.

Unlike v0.01 — which was produced by the `cedar-c2v-cpp` `cef-custom-build/`
pipeline — v0.1.0 was built directly from the `cedar-render-foundation`
source repository, which keeps the CEF / Chromium core source and its own
build scripts (`scripts/prepare-build.sh`, `scripts/build.sh`).

## What's included

v0.1.0 keeps the standard CEF `minimal` distribution layout.

The top-level directory inside the tarball remains:

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
```

Main contents:

```
cef_binary_..._linux64_minimal/
├── CMakeLists.txt
├── include/
├── cmake/
├── libcef_dll/
├── Release/
│   └── libcef.so
└── Resources/
```

Key parts:

- `Release/libcef.so`: the main Chromium / CEF runtime library.
- `Resources/`: Chromium / CEF runtime resources.
- `include/`: CEF C/C++ headers.
- `libcef_dll/`: source for `libcef_dll_wrapper`.
- `cmake/` and `CMakeLists.txt`: CMake integration files for consuming the CEF distribution.

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

v0.1.0 is a clean Cedar-built CEF `minimal` distribution of the pinned
`152.0.7977.83` baseline. It does **not** add any extra native export beyond
standard CEF.

In particular, this release does **not** export:

```
cef_request_raw_snapshot
```

That Cedar-private raw-snapshot capability — present in v0.01 — is **not**
part of this build. A consumer that requires the raw-snapshot symbol must use
a foundation release that provides it (such as v0.01), not v0.1.0.

The exported symbol table of this release's `libcef.so` matches a standard CEF
`152.0.7977.83` minimal build.

## Using it in cedar-c2v-cpp

The distribution follows the standard CEF `minimal` layout, so an existing CEF
CMake integration can consume it the same way:

```
cedar-render-foundation-dist v0.1.0
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
runtime uses this libcef.so
```

> **Compatibility note.** `cedar-c2v-cpp` `v2.1.2` expects the raw-snapshot
> capability and fails the production build if `cef_request_raw_snapshot` is
> missing. Because v0.1.0 does not export that symbol, it is **not** a
> drop-in replacement for v0.01 under that gated build. Use v0.1.0 only where
> the raw-snapshot capability is not required, or adjust the consumer's
> capability gate accordingly.

## Artifact identity

Verified artifact:

| Item | Value |
| --- | --- |
| package size | `321,479,861` bytes |
| package SHA-256 | `12e52fe818053a4e837a0c87b410cb22d4018b62096bb020439f71618b6a261a` |
| `libcef.so` SHA-256 | `8704a8783110e906a8ece432541064036f29c090ac04eb4485440c886b5f5579` |
| custom export | none (standard CEF export set) |

This artifact identity is distinct from v0.01. v0.1.0 is a separate rebuild
with different byte contents, so its SHA-256 values do not match v0.01 and must
not be substituted for it.

If the archive is repacked, modified, or rebuilt in the future, the artifact
identity must be recalculated and recorded even if the release version stays
the same.

## Functional verification

The published package was verified with the foundation repository's
`render_check` out-of-tree consumer, which extracts this `minimal` package,
builds a minimal CEF program against it, and renders a local HTML page using
off-screen rendering with software compositing (no X server required):

```
painted=1  red=1 green=1 blue=1  timed_out=0  exit code 0
```

The red / green / blue marker colors were all hit, confirming the package can
be consumed and renders HTML correctly.

## Consumer

At the product level, consumers depend on:

```
Cedar version + Chromium baseline
```

For this release:

```
v0.1.0 + Chromium 152.0.7977.83
```
