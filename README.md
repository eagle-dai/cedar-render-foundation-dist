# cedar-render-foundation-dist

Prebuilt distribution repository for the Cedar Render foundation layer.

This repository does not contain Chromium / CEF implementation source code. It stores and publishes compiled and verified distributions for upper-layer projects such as `cedar-c2v-cpp`.

## Releases

| Version | Chromium | Package | Platform | Details |
| --- | --- | --- | --- | --- |
| v0.2.0 | `152.0.7977.83` | `cedar-render-foundation-v0.2.0-chromium-152.0.7977.83-linux-x64.tar.bz2` | Linux x64 | [release notes](releases/v0.2.0/README.md) |
| v0.1.0 | `152.0.7977.83` | `cedar-render-foundation-v0.1.0-chromium-152.0.7977.83-linux-x64.tar.bz2` | Linux x64 | [release notes](releases/v0.1.0/README.md) |
| v0.01 | `152.0.7977.83` | `cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2` | Linux x64 | [release notes](releases/v0.01/README.md) |

More detailed information such as the CEF version, SHA-256 values, and custom capabilities is kept under `releases/`.

## Repository role

```
render foundation implementation
        │
        │ build
        ▼
cedar-render-foundation-dist
        │
        │ prebuilt distribution
        ▼
cedar-c2v-cpp
```

- Renderer / Chromium / CEF implementation and research: upstream foundation implementation repository
- This repository: publishes verified prebuilt distributions
- `cedar-c2v-cpp`: consumes the distribution and performs video generation

## Package naming

Release packages include both the Cedar release version and the Chromium version:

```
cedar-render-foundation-v<version>-chromium-<chromium-version>-<platform>-<arch>.tar.bz2
```

Example:

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

This makes both the Cedar release and the Chromium baseline visible from the filename. The exact CEF version is recorded in the corresponding release notes.
