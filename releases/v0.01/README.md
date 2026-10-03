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

Overall flow:

```
Cedar distribution tarball
        │
        │ stage
        ▼
cedar-c2v-cpp/cef/third_party/cef/
        │
        │ Docker COPY
        ▼
CEF_ROOT
        │
        ▼
build cefservice / framemerger
        │
        ▼
runtime uses Cedar libcef.so
```

### 1. Download the package

Download:

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

No manual extraction is required.

### 2. Stage it into cedar-c2v-cpp

`cedar-c2v-cpp` already provides:

```
cef-custom-build/scripts/stage-cef-artifact.sh
```

The script:

1. checks that the archive is safe;
2. checks that the archive has the expected top-level directory;
3. verifies `Release/`, `Resources/`, `include/`, `cmake/`, and `libcef_dll/`;
4. verifies `Release/libcef.so`;
5. uses `nm` to confirm that `cef_request_raw_snapshot` is exported;
6. calculates SHA-256 for both the tarball and `libcef.so`;
7. stages the validated distribution under:

```
cef/third_party/cef/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
```

### 3. Current v0.01 filename compatibility

The current `cedar-c2v-cpp` `stage-cef-artifact.sh` still validates the archive basename and expects the original CEF filename:

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

Until the consumer is updated to accept the Cedar package name directly, create a symlink instead of copying the roughly 321 MB file:

```bash
ln -s \
  /absolute/path/to/cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2 \
  /tmp/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2

export CEF_ARTIFACT="/tmp/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2"

bash cef-custom-build/scripts/stage-cef-artifact.sh
```

Only the external filename changes. The directory inside the tarball still follows the standard CEF distribution layout, so the CMake integration does not need to change.

Once `cedar-c2v-cpp` accepts Cedar package names directly, this compatibility step can be removed.

### 4. Build cedar-c2v-cpp

After staging:

```bash
docker build --shm-size=1g \
  -t videoengine \
  -f docker/Dockerfile.service .
```

The Docker build copies:

```
cef/third_party/cef/
```

into the build image.

When the CEF CMake logic sees that the required distribution is already present, it uses it as `CEF_ROOT` instead of downloading the official CEF package.

The default production build is fail-closed:

> If the staged Cedar CEF distribution is missing, or if `libcef.so` does not export `cef_request_raw_snapshot`, the build fails instead of silently falling back to official CEF.

Only an explicit:

```bash
--build-arg ALLOW_OFFICIAL_CEF=1
```

allows an official CEF / CDP control build.

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
