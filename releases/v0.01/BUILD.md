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
v2.1.2
```

That tag contains the custom CEF source-build flow, raw-snapshot patch, version lock, and build documentation used as the reproducible reference for this release.

## Pinned inputs

| Item | Value |
| --- | --- |
| CEF version | `152.0.6+g708dc14+chromium-152.0.7977.83` |
| CEF branch | `7977` |
| Chromium | `152.0.7977.83` |
| Chromium checkout | `refs/tags/152.0.7977.83` |
| cedar-c2v-cpp reference tag | `v2.1.2` |
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
record / verify artifact identity
(size + SHA-256)
        │
        ▼
rename archive for Cedar release
        │
        ▼
publish as v0.01 asset
```

### 1. Prepare a build host

Use a Linux x64 environment that can build Chromium / CEF.

Before running the repository build script, the host needs the basic bootstrap tools used before Chromium's own dependency installer can run:

- `bash`
- `git`
- `curl`
- `python3`
- standard archive/checksum tools such as `tar` and `sha256sum`
- permission to install the Linux packages requested by Chromium's `install-build-deps.sh`

The build needs enough disk space and memory for a full Chromium / CEF source build, but this release does not require a specific cloud instance type, CPU count, RAM size, or storage layout.

The source checkout and build cache should live outside the Git repository.

Example:

```bash
WORK_DIR=/data/cef-build
```

### 2. Run the source-build automation

Use the `cedar-c2v-cpp` tag recorded for this release:

```bash
git checkout v2.1.2

WORK_DIR=/data/cef-build \
  INSTALL_TO_REPO=0 \
  bash cef-custom-build/scripts/build-cef-from-source.sh
```

`INSTALL_TO_REPO=0` is recommended when the goal is only to produce the release artifact. It leaves the generated CEF distribution under `WORK_DIR` instead of also extracting a copy into `cedar-c2v-cpp/cef/third_party/cef/`.

The script performs four important build phases:

1. Sync the pinned CEF/Chromium source without building or packaging.
2. Apply the Cedar raw-snapshot patch.
3. Install the Chromium build dependencies from the checked-out source tree.
4. Build `libcef` and create the CEF `minimal` distribution.

The exact checkout and tool revisions are intentionally not duplicated here. They are read from `cef-custom-build/CEF_VERSION.lock` in the `cedar-c2v-cpp` `v2.1.2` tag.

The final build/package phase uses the pinned CEF driver to build `libcef`, preserve the already-applied Cedar patch, and generate the Linux x64 Release `minimal` distribution.

### 3. Produced CEF archive

The build produces:

```
/data/cef-build/chromium/src/cef/binary_distrib/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

The archive keeps the normal CEF `minimal` distribution layout.

### 4. Verify the Cedar capability

The build script refuses to accept a built `libcef.so` that does not export the Cedar API.

Equivalent manual check for the built library:

```bash
nm -D --defined-only \
  /data/cef-build/chromium/src/out/Release_GN_x64/libcef.so \
  | grep cef_request_raw_snapshot
```

Also verify the copy that is actually inside the generated distribution:

```bash
CEF_ARCHIVE=/data/cef-build/chromium/src/cef/binary_distrib/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
TMP_DIR="$(mktemp -d)"

tar -xjf "$CEF_ARCHIVE" -C "$TMP_DIR"

nm -D --defined-only \
  "$TMP_DIR/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/Release/libcef.so" \
  | grep cef_request_raw_snapshot

rm -rf "$TMP_DIR"
```

Both checks must find:

```
cef_request_raw_snapshot
```

### 5. Record the published artifact identity

For reference, the originally published v0.01 artifact was:

| Item | Value |
| --- | --- |
| original archive | `cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2` |
| size | `321,478,158` bytes |
| archive SHA-256 | `7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1` |
| `libcef.so` SHA-256 | `d8bb85ad84bf5eaee62b961f6d255aaf20a7db5db8ca02a14ca684c4df4aad53` |
| required export | `cef_request_raw_snapshot` |

### 6. Rename for the Cedar release

The generated CEF archive is:

```
/data/cef-build/chromium/src/cef/binary_distrib/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

Publish the generated archive under the Cedar-facing name:

```bash
cp \
  /data/cef-build/chromium/src/cef/binary_distrib/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2 \
  cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2

sha256sum cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

Do not extract and repack the archive. Only the external filename changes.

For reference, the originally published v0.01 asset has SHA-256:

```
7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1
```

This checksum identifies the published asset only. It is not a rebuild acceptance criterion.

If the SHA-256 differs from the published v0.01 asset, do **not** replace the existing v0.01 binary under the same release. Publish the rebuilt binary as a new foundation release and update the consuming `cedar-c2v-cpp` release URL / SHA-256 explicitly.

If exact byte identity with v0.01 is required, use the existing v0.01 release asset and verify its recorded SHA-256 instead of relying on a fresh Chromium / CEF rebuild.

## Rebuilding from the same inputs

For the closest reproduction of this release:

1. Obtain `cedar-c2v-cpp` and check out tag `v2.1.2`.
2. Ensure the basic bootstrap tools above are available.
3. Use the pinned values in `cef-custom-build/CEF_VERSION.lock`.
4. Run `cef-custom-build/scripts/build-cef-from-source.sh`, preferably with `INSTALL_TO_REPO=0` when producing only the release artifact.
5. Verify `cef_request_raw_snapshot` exists in the built and packaged `libcef.so`.
6. Copy the generated tarball to the Cedar-facing filename without repacking it.
7. Validate that the package can be consumed by `cedar-c2v-cpp` as expected.

The goal of reproduction is to rebuild the same Cedar-customized CEF configuration and behavior from the pinned source/build inputs.

The v0.01 checksum is kept only to identify the originally published binary asset.
