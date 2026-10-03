# How v0.01 was built

This document records how the binary asset in this release was produced.

The important point is:

> `cedar-render-foundation-dist` does not build Chromium / CEF itself.  
> v0.01 was produced by the custom CEF source-build pipeline in `cedar-c2v-cpp/cef-custom-build/`, then the verified `minimal` distribution was published here.

## Source of truth

The build procedure comes from `cedar-c2v-cpp`:

- `cef-custom-build/cef-source-build.md`
- `cef-custom-build/CEF_VERSION.lock`
- `cef-custom-build/scripts/build-cef-from-source.sh`
- `cef-custom-build/patches/0001-b1-native-raw-snapshot.patch`

For v0.01, use the stable `cedar-c2v-cpp` tag:

```
v2.0.0
```

That tag contains the custom CEF source-build flow, raw-snapshot patch, version lock, and build documentation used as the reproducible reference for this release.

## Pinned inputs

| Item | Value |
| --- | --- |
| CEF version | `152.0.6+g708dc14+chromium-152.0.7977.83` |
| CEF branch | `7977` |
| Chromium | `152.0.7977.83` |
| Chromium checkout | `refs/tags/152.0.7977.83` |
| cedar-c2v-cpp reference tag | `v2.0.0` |
| Exact source/tool revisions | see `cef-custom-build/CEF_VERSION.lock` in that tag |
| Distribution | Linux x64, Release, CEF `minimal` |
| Archive format | `tar.bz2` |

Build arguments:

```
is_official_build=true
use_sysroot=true
symbol_level=1
is_cfi=false
```

The build uses PGO profiles and the official CEF `automate-git.py` driver pinned to the selected CEF revision.

## Cedar customization

The only intentional CEF source customization for this release is:

```
cef-custom-build/patches/0001-b1-native-raw-snapshot.patch
```

It adds the project-private export:

```
cef_request_raw_snapshot
```

The implementation reuses Chromium's internal:

```
RenderWidgetHostImpl::GetSnapshotFromBrowser(from_surface=true)
```

to obtain raw BGRA pixels before the CDP screenshot JPEG encode/decode path.

## Build flow

The release is generated as follows:

```
pinned CEF / Chromium source
        │
        ├─ pinned depot_tools
        ├─ pinned automate-git.py
        └─ PGO profiles
        │
        ▼
sync source only
        │
        ▼
apply 0001-b1-native-raw-snapshot.patch
        │
        ▼
install Chromium build dependencies
        │
        ▼
build libcef + chrome_sandbox
        │
        ▼
package CEF minimal distribution
        │
        ▼
verify cef_request_raw_snapshot
        │
        ▼
verify artifact SHA-256
        │
        ▼
rename archive for Cedar release
        │
        ▼
publish as v0.01 asset
```

### 1. Prepare a build host

The reference build host recorded in `cedar-c2v-cpp` was:

- Ubuntu
- EC2 `c5.4xlarge`
- 16 vCPU
- 30 GiB RAM
- dedicated storage of at least 250 GiB
- swap enabled for the ThinLTO link step

The source checkout and build cache live outside the Git repository.

Example:

```bash
WORK_DIR=/data/cef-build
```

### 2. Run the source-build automation

From the `cedar-c2v-cpp` repository:

```bash
WORK_DIR=/data/cef-build \
  bash cef-custom-build/scripts/build-cef-from-source.sh
```

The script performs four important build phases:

1. Sync the pinned CEF/Chromium source without building or packaging.
2. Apply the Cedar raw-snapshot patch.
3. Install the Chromium build dependencies from the checked-out source tree.
4. Build `libcef` and create the CEF `minimal` distribution.

The exact checkout and tool revisions are intentionally not duplicated here. They are read from `cef-custom-build/CEF_VERSION.lock` in the `cedar-c2v-cpp` `v2.0.0` tag.

The final build/package phase uses the pinned CEF driver to build `libcef`, preserve the already-applied Cedar patch, and generate the Linux x64 Release `minimal` distribution.

### 3. Produced CEF archive

The build produces:

```
/data/cef-build/chromium/src/cef/binary_distrib/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

The archive keeps the normal CEF `minimal` distribution layout.

### 4. Verify the Cedar capability

The build script refuses to accept a `libcef.so` that does not export the Cedar API.

Equivalent manual check:

```bash
nm -D --defined-only \
  /data/cef-build/chromium/src/out/Release_GN_x64/libcef.so \
  | grep cef_request_raw_snapshot
```

The packaged copy must also contain the same export in:

```
Release/libcef.so
```

### 5. Verify artifact identity

The artifact used for v0.01 was verified as:

| Item | Value |
| --- | --- |
| original archive | `cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2` |
| size | `321,478,158` bytes |
| archive SHA-256 | `7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1` |
| `libcef.so` SHA-256 | `d8bb85ad84bf5eaee62b961f6d255aaf20a7db5db8ca02a14ca684c4df4aad53` |
| required export | `cef_request_raw_snapshot` |

### 6. Rename for the Cedar release

The verified CEF archive was published under the Cedar-facing name:

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

Only the filename changed. The archive bytes were not repacked or modified.

Therefore the published asset keeps the same SHA-256:

```
7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1
```

## Reproducing v0.01

For the closest reproduction of this release:

1. Check out the `cedar-c2v-cpp` tag `v2.0.0`.
2. Use the pinned values in `cef-custom-build/CEF_VERSION.lock`.
3. Run `cef-custom-build/scripts/build-cef-from-source.sh`.
4. Verify `cef_request_raw_snapshot` exists.
5. Record the new tarball and `libcef.so` SHA-256 values.
6. Compare them with the v0.01 identity above.

The source inputs are pinned, but the Chromium/CEF build is not guaranteed to be bit-for-bit reproducible across different hosts. A rebuilt archive may therefore have a different SHA-256 even when the same source revisions and build parameters are used.

The v0.01 SHA-256 identifies the exact binary published in this release; the pinned source revisions, patch, build arguments, and build script provide its provenance.
