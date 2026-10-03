# cedar-render-foundation-dist

Cedar Render 基础渲染层的**预编译 distribution 发布仓库**。

本仓库**不存放 Chromium / CEF 实现源码**，只发布已经编译并验证过的 distribution，供 `cedar-c2v-cpp` 等上层项目直接使用。

当前 Linux x64 的发布格式保持 CEF 标准 `minimal` distribution 结构，不再额外重新打包。这样现有 `cedar-c2v-cpp` 可以直接消费，无需修改它的 CMake/CEF 集成方式。

简单理解：

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

## 当前包

当前目标版本：

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

对应：

- CEF: `152.0.6+g708dc14`
- CEF commit: `708dc140cbc3286826a8abef89dc23a44ff9ea72`
- Chromium: `152.0.7977.83`
- Platform: Linux x64
- Distribution: `minimal`

这不是官方原版 CEF binary。

其中的 `libcef.so` 带有 Cedar 使用的定制能力：

```
cef_request_raw_snapshot
```

`cedar-c2v-cpp` 会在 build 和 runtime 两个阶段检查/使用这个能力。

## 包里面有什么

这是完整的 CEF minimal binary distribution，解压后顶层类似：

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
├── CMakeLists.txt
├── include/
├── cmake/
├── libcef_dll/
├── Release/
│   └── libcef.so
└── Resources/
```

因此它同时包含：

- C/C++ headers
- `libcef.so`
- `libcef_dll_wrapper` 的构建源码/配置
- CEF CMake 配置
- Chromium runtime resources

这里的 `dist` 指**可供上层 C/C++ 项目直接集成的预编译 renderer / CEF distribution**，不是渲染基础层的实现源码。

## cedar-c2v-cpp 如何使用

`cedar-c2v-cpp` 已经支持直接使用这个 tarball。

### 1. 下载 release package

从本仓库的 GitHub Releases 下载：

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

不需要手工解压。

### 2. Stage 到 cedar-c2v-cpp

在 `cedar-c2v-cpp` 根目录执行：

```bash
export CEF_ARTIFACT="/absolute/path/to/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2"

bash cef-custom-build/scripts/stage-cef-artifact.sh
```

这个脚本不是简单的 `tar -x`。

它会先检查：

- 文件名是否和 `cef-custom-build/CEF_VERSION.lock` 一致；
- archive 是否只有正确的顶层目录；
- 是否包含 `Release/`、`Resources/`、`include/`、`cmake/`、`libcef_dll/`；
- 是否存在 `Release/libcef.so`；
- `libcef.so` 是否真的导出 `cef_request_raw_snapshot`。

验证通过后，包会被放到：

```
cedar-c2v-cpp/
└── cef/
    └── third_party/
        └── cef/
            └── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
```

### 3. 正常构建 cedar-c2v-cpp

然后直接：

```bash
docker build --shm-size=1g \
  -t videoengine \
  -f docker/Dockerfile.service .
```

不需要重新编译 Chromium / CEF。

## 内部是怎么接上的

调用链如下：

```
cedar-render-foundation-dist release
        │
        │ download tar.bz2
        ▼
stage-cef-artifact.sh
        │
        ▼
cef/third_party/cef/
cef_binary_<version>_linux64_minimal/
        │
        │ Docker COPY
        ▼
/app/cef/third_party/cef/
        │
        ▼
cef/cmake/DownloadCEF.cmake
        │
        │ directory already exists
        │ → skip official CEF download
        ▼
CEF_ROOT
        │
        ├── include headers
        ├── build libcef_dll_wrapper
        ├── link Release/libcef.so
        └── copy CEF runtime/resources
        ▼
cefservice
```

关键点是：

> `DownloadCEF.cmake` 本来就先检查目标 CEF directory 是否已经存在。

如果 staged package 已经存在，它不会访问 `cef-builds.spotifycdn.com` 下载官方 CEF，而是直接把这个目录设为 `CEF_ROOT`。

所以这个 distribution 可以在**不修改 cedar-c2v-cpp 原有 CEF CMake 集成逻辑**的情况下替换官方 CEF。

## Build 时的保护

`docker/Dockerfile.service` 默认是 fail-closed。

CMake configure 之前，`check-staged-cef.sh` 会确认对应版本已经 staged。

CMake configure 之后还会再次检查：

```bash
nm -D --defined-only Release/libcef.so
```

必须能找到：

```
cef_request_raw_snapshot
```

否则默认 production build 直接失败，避免不小心退回官方 CEF 的 CDP-only 路径。

只有显式使用：

```bash
--build-arg ALLOW_OFFICIAL_CEF=1
```

才允许使用官方 CEF 作为 CDP control build。

## Runtime 选择

`cedar-c2v-cpp` 运行时通过 `dlsym` 检测当前加载的 `libcef.so` 是否提供：

```
cef_request_raw_snapshot
```

`VG_RAW_SNAPSHOT` 的行为：

| 设置 | 行为 |
| --- | --- |
| 未设置 | libcef 支持 raw → raw；不支持 → CDP |
| `VG_RAW_SNAPSHOT=0` | 强制 CDP |
| `VG_RAW_SNAPSHOT=1` | 强制要求 raw；libcef 不支持则失败 |

正常 production build 不需要设置它；定制 `libcef.so` 被正确加载后会自动走 raw path。

## 已验证 artifact identity

当前在 `cedar-c2v-cpp` 中记录并完成 end-to-end 验证的 patched artifact：

| 项目 | 值 |
| --- | --- |
| filename | `cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2` |
| size | `321,478,158` bytes |
| tarball SHA-256 | `7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1` |
| `libcef.so` SHA-256 | `d8bb85ad84bf5eaee62b961f6d255aaf20a7db5db8ca02a14ca684c4df4aad53` |
| custom export | `cef_request_raw_snapshot` |

注意：CEF / Chromium source build 不保证不同机器之间 bit-for-bit reproducible。SHA-256 用于识别这个已经验证过的具体 artifact，不代表所有从相同源码重新编译的文件都必须得到相同 hash。

## 三个仓库的职责

### cedar-render-shell

当前负责研究和实现 Cedar 的 Chromium / CEF 底层渲染路线，包括 frame identity、compositor output 和更直接的 pixel delivery。

它是**实现与研究仓库**，不是 distribution 仓库。

### cedar-render-foundation-dist

**本仓库。**

负责保存和发布由 Cedar 基础渲染层产出的、已经编译并验证过的 distribution。

它的职责是：

- 发布预编译 CEF / renderer artifact；
- 固定版本与 artifact identity；
- 记录 SHA-256 和 custom capabilities；
- 让上层项目无需重新编译 Chromium / CEF 即可使用。

### cedar-c2v-cpp

最终的视频生成项目。

它负责：

- 调用 renderer；
- worker / concurrency；
- frame processing；
- AV1 / WebM 等视频编码与输出；
- 产品级 end-to-end 流程。

整体关系：

```
cedar-render-shell
  implementation / research
        │
        │ build
        ▼
cedar-render-foundation-dist
        │
        │ verified prebuilt distribution
        ▼
cedar-c2v-cpp
        │
        ▼
video
```

## 发布约定

每个 Release 至少记录：

- package filename
- distribution version
- CEF version / commit
- Chromium version
- target platform / architecture
- 对应实现 / patch 的来源 commit
- tarball SHA-256
- `libcef.so` SHA-256
- custom exports / capabilities

目标是让 `cedar-c2v-cpp` 只需要：

> 下载一个经过验证的 distribution，而不需要重新编译 Chromium。
