# Cedar Render Foundation v0.01

The first Cedar Render Foundation distribution.

Its purpose is simple:

> Provide `cedar-c2v-cpp` with a compiled and verified Cedar-customized Chromium / CEF foundation package, so Chromium / CEF does not need to be rebuilt for every normal build.

## Package

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

## Base

This release is based on:

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

| Item | Value |
| --- | --- |
| Cedar version | `v0.01` |
| CEF | `152.0.6+g708dc14` |
| Chromium | `152.0.7977.83` |
| Platform | Linux x64 |
| Distribution | CEF `minimal` |

## How this release was built

See [`BUILD.md`](BUILD.md) for the build provenance and reproduction flow.

It documents the `cedar-c2v-cpp` `v2.1.2` custom CEF source-build pipeline used as the reference for v0.01: version lock, Cedar raw-snapshot patch, build arguments, packaging, export verification, checksums, and the final release rename.

## What's included

v0.01 keeps the standard CEF `minimal` distribution layout.

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

- `Release/libcef.so`: the main Chromium / CEF runtime library and the location of the Cedar-specific capability.
- `Resources/`: Chromium / CEF runtime resources.
- `include/`: CEF C/C++ headers.
- `libcef_dll/`: source for `libcef_dll_wrapper`.
- `cmake/` and `CMakeLists.txt`: CMake integration files for consuming the CEF distribution.

This package is therefore more than a single `libcef.so`. It is a complete CEF minimal distribution that can be consumed directly by an existing CEF CMake project.

### What's not included

This package does not include:

- `cedar-c2v-cpp` application code
- the Node.js HTML-to-video service
- framemerger
- AV1 / WebM encoder code
- Cedar upper-layer worker / scheduling logic

Those remain part of `cedar-c2v-cpp`.

## Difference from official CEF

v0.01 is not an unmodified official CEF binary.

Cedar adds the following export to `libcef.so`:

```
cef_request_raw_snapshot
```

This native C export allows `cedar-c2v-cpp` to request a raw pixel snapshot directly instead of relying only on the Chrome DevTools Protocol (CDP) screenshot path.

At runtime, `cedar-c2v-cpp` uses `dlsym` to detect whether this symbol is available, so it does not need a hard link-time dependency on the Cedar-private API.

Runtime selection rules:

| `VG_RAW_SNAPSHOT` | Behavior |
| --- | --- |
| unset | use raw if the Cedar capability is available; otherwise use CDP |
| `0` | force CDP |
| `1` | require raw; fail if the capability is unavailable |

Normal production use does not require setting `VG_RAW_SNAPSHOT`.

## Using it in cedar-c2v-cpp

For `cedar-c2v-cpp v2.1.2`, normal builds consume this release automatically.

Overall flow:

```
cedar-render-foundation-dist v0.01
        │
        │ DownloadCEF.cmake
        │ + SHA-256 verification
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
runtime uses Cedar libcef.so
```

### Normal build

From `cedar-c2v-cpp`:

```bash
docker build --shm-size=1g \
  -t videoengine \
  -f docker/Dockerfile.service .
```

No manual download, rename, symlink, or staging command is required.

`cef/cmake/DownloadCEF.cmake` points to this v0.01 release asset and pins its SHA-256.

If the matching extracted CEF distribution already exists under:

```
cef/third_party/cef/
```

it is reused and no download occurs.

Otherwise CMake downloads the published v0.01 package, verifies its SHA-256, and extracts the standard CEF `minimal` distribution.

### Raw-capability gate

The Docker build checks that:

```
Release/libcef.so
```

exports:

```
cef_request_raw_snapshot
```

If the symbol is missing, the production build fails instead of silently using a non-raw CEF distribution.

### When the source build is needed

The slow Chromium / CEF source build is not part of a normal `cedar-c2v-cpp` build.

Run it only when producing a new foundation binary, for example when:

- upgrading CEF / Chromium;
- changing the Cedar raw-snapshot patch;
- changing the CEF build arguments.

See `BUILD.md` and the `cedar-c2v-cpp v2.1.2` files under `cef-custom-build/` for that process.

## Why a full distribution is published

In theory, only the modified `libcef.so` could be published.

v0.01 publishes the complete CEF minimal distribution instead because:

- the existing `cedar-c2v-cpp` CMake integration can consume it directly;
- headers, wrapper sources, CMake metadata, and runtime resources stay aligned with the CEF version;
- the consumer does not need a special "replace libcef.so" workflow;
- version locking and integrity checks are simpler.

In other words, v0.01 does not try to redesign CEF packaging. Its goal is:

> Provide a directly usable Cedar-customized CEF distribution while changing as little as possible in the existing `cedar-c2v-cpp` CEF integration.

## Artifact identity

Verified artifact:

| Item | Value |
| --- | --- |
| package size | `321,478,158` bytes |
| package SHA-256 | `7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1` |
| `libcef.so` SHA-256 | `d8bb85ad84bf5eaee62b961f6d255aaf20a7db5db8ca02a14ca684c4df4aad53` |
| custom export | `cef_request_raw_snapshot` |

v0.01 only renames the already verified CEF tarball to the Cedar-facing package name. The file contents do not change, so the SHA-256 remains unchanged.

If the archive is repacked, modified, or rebuilt in the future, the artifact identity must be recalculated and recorded even if the release version stays the same.

## Consumer

Current primary consumer:

- `cedar-c2v-cpp`

At the product level, consumers depend on:

```
Cedar version + Chromium baseline
```

For this release:

```
v0.01 + Chromium 152.0.7977.83
```

The CEF version and build details remain in this release note for traceability and reproducibility. Exact source/tool revisions stay in the `cedar-c2v-cpp` `v2.1.2` version lock rather than being duplicated here.
