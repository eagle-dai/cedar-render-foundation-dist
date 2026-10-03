# cedar-render-foundation-dist

Cedar Render 基础渲染层的预编译发行仓库。

本仓库不保存 Chromium / CEF 实现源码，只保存和发布已经编译、验证过的 distribution，供 `cedar-c2v-cpp` 等上层项目直接使用。

## Releases

| Version | Chromium | Package | Platform | Details |
| --- | --- | --- | --- | --- |
| v0.01 | `152.0.7977.83` | `cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2` | Linux x64 | [release notes](releases/v0.01/README.md) |

CEF 版本、SHA-256、custom capabilities 等更具体的信息统一放在 `releases/` 下。

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

- renderer / Chromium / CEF 的实现与研究：上游 foundation 实现仓库
- 本仓库：发布经过验证的预编译 distribution
- `cedar-c2v-cpp`：消费 distribution 并完成视频生成

## Package naming

发行包同时包含 Cedar release version 和 Chromium version：

```
cedar-render-foundation-v<version>-chromium-<chromium-version>-<platform>-<arch>.tar.bz2
```

例如：

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

这样可以同时看出 Cedar 发行版本和底层 Chromium 基线；CEF 的具体版本记录在对应 release notes 中。
