# Cedar Render Foundation v0.01

第一版 Cedar Render Foundation distribution。

## Package

```
cedar-render-foundation-v0.01-linux-x64.tar.bz2
```

## Base

该版本基于：

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

| Item | Value |
| --- | --- |
| Cedar version | `v0.01` |
| CEF | `152.0.6+g708dc14` |
| CEF commit | `708dc140cbc3286826a8abef89dc23a44ff9ea72` |
| Chromium | `152.0.7977.83` |
| Platform | Linux x64 |
| Distribution | CEF `minimal` |

## Cedar customization

该版本不是官方原版 CEF binary。

`libcef.so` 增加：

```
cef_request_raw_snapshot
```

用于 Cedar 的 raw snapshot 路径。

## Artifact identity

当前已验证 artifact：

| Item | Value |
| --- | --- |
| original artifact size | `321,478,158` bytes |
| original tarball SHA-256 | `7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1` |
| `libcef.so` SHA-256 | `d8bb85ad84bf5eaee62b961f6d255aaf20a7db5db8ca02a14ca684c4df4aad53` |

> 注意：如果只是把原始 tarball 改名为 Cedar package，文件内容不变，SHA-256 也不会改变；如果重新打包，则必须重新记录新的 package SHA-256。

## Consumer

主要消费方：

- `cedar-c2v-cpp`

上层项目应优先依赖 Cedar release version，而不是直接依赖 CEF / Chromium 的长版本号。
