# cedar-render-foundation-dist

Cedar Render 基础渲染层的预编译发行仓库。

本仓库不保存 Chromium / CEF 实现源码，只保存和发布已经编译、验证过的 distribution，供 `cedar-c2v-cpp` 等上层项目直接使用。

## Releases

| Version | Package | Platform | Details |
| --- | --- | --- | --- |
| v0.01 | `cedar-render-foundation-v0.01-linux-x64.tar.bz2` | Linux x64 | [release notes](releases/v0.01/README.md) |

版本相关的 Chromium / CEF 信息、SHA-256、custom capabilities 等细节统一放在 `releases/` 下，不放在首页 README。

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

发行包使用 Cedar 自己的版本号命名：

```
cedar-render-foundation-v<version>-<platform>-<arch>.tar.bz2
```

例如：

```
cedar-render-foundation-v0.01-linux-x64.tar.bz2
```

底层 CEF / Chromium 的具体版本属于 release implementation detail，记录在对应版本目录中。
