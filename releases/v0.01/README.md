# Cedar Render Foundation v0.01

第一版 Cedar Render Foundation distribution。

它的目标很简单：

> 给 `cedar-c2v-cpp` 提供一个已经编译、验证过的 Cedar 定制 Chromium / CEF 基础运行包，避免每次都重新编译 Chromium / CEF。

## Package

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
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

## 包里有什么

v0.01 保持 CEF 标准 `minimal` distribution 结构。

tarball 内部的顶层目录仍然是：

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
```

主要内容：

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

其中：

- `Release/libcef.so`：Chromium / CEF 的主要运行库，也是 Cedar 定制能力所在。
- `Resources/`：Chromium / CEF 运行时需要的资源文件。
- `include/`：CEF C/C++ headers。
- `libcef_dll/`：`libcef_dll_wrapper` 的源码。
- `cmake/`、`CMakeLists.txt`：供 CMake 项目直接集成 CEF。

因此这个包不仅是一个 `libcef.so`，而是一个可以直接被现有 CEF CMake 工程消费的完整 minimal distribution。

### 不包含什么

这个包不包含：

- `cedar-c2v-cpp` 的业务代码；
- HTML → video 的 Node.js 服务；
- framemerger；
- AV1 / WebM encoder；
- Cedar 的上层 worker / scheduling 逻辑。

这些仍然属于 `cedar-c2v-cpp`。

## 和官方 CEF 的区别

v0.01 不是官方原版 CEF binary。

Cedar 在 `libcef.so` 中增加了：

```
cef_request_raw_snapshot
```

这个 native C export 让 `cedar-c2v-cpp` 可以直接请求 raw pixel snapshot，而不必只依赖 Chrome DevTools Protocol（CDP）截图路径。

`cedar-c2v-cpp` 在运行时通过 `dlsym` 检查这个 symbol 是否存在，因此不需要在链接阶段硬依赖 Cedar 私有 API。

运行时选择规则：

| `VG_RAW_SNAPSHOT` | 行为 |
| --- | --- |
| 未设置 | 有 Cedar raw capability → raw；否则 → CDP |
| `0` | 强制使用 CDP |
| `1` | 强制要求 raw；没有该 capability 则失败 |

正常 production 使用不需要设置 `VG_RAW_SNAPSHOT`。

## 在 cedar-c2v-cpp 中如何使用

整体流程：

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

### 1. 下载 package

下载：

```
cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2
```

不需要手工解压。

### 2. Stage 到 cedar-c2v-cpp

`cedar-c2v-cpp` 已有：

```
cef-custom-build/scripts/stage-cef-artifact.sh
```

它会：

1. 检查 archive 是否安全；
2. 检查 archive 顶层目录是否正确；
3. 检查 `Release/`、`Resources/`、`include/`、`cmake/`、`libcef_dll/`；
4. 检查 `Release/libcef.so`；
5. 用 `nm` 确认 `cef_request_raw_snapshot` 确实存在；
6. 计算 tarball 和 `libcef.so` 的 SHA-256；
7. 验证全部通过后再放入：

```
cef/third_party/cef/
└── cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal/
```

### 3. v0.01 当前的文件名兼容处理

当前 `cedar-c2v-cpp` 的 `stage-cef-artifact.sh` 仍然会检查 archive basename，要求旧的 CEF 文件名：

```
cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2
```

因此，在 consumer 还没有更新为原生识别 Cedar package name 前，可以建立一个 symlink，不需要复制约 321 MB 的文件：

```bash
ln -s \
  /absolute/path/to/cedar-render-foundation-v0.01-chromium-152.0.7977.83-linux-x64.tar.bz2 \
  /tmp/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2

export CEF_ARTIFACT="/tmp/cef_binary_152.0.6+g708dc14+chromium-152.0.7977.83_linux64_minimal.tar.bz2"

bash cef-custom-build/scripts/stage-cef-artifact.sh
```

这里仅改变外部文件名；tarball 内部仍保持标准 CEF distribution directory，所以 CMake 集成不需要改变。

后续 `cedar-c2v-cpp` 应直接接受 Cedar package name，这个兼容步骤即可删除。

### 4. 构建 cedar-c2v-cpp

完成 stage 后，正常构建：

```bash
docker build --shm-size=1g \
  -t videoengine \
  -f docker/Dockerfile.service .
```

Docker build 会把：

```
cef/third_party/cef/
```

复制到 build image。

CEF CMake 逻辑发现对应 distribution 已经存在后，会直接将其作为 `CEF_ROOT` 使用，不再下载官方 CEF。

默认 production build 是 fail-closed：

> 如果没有找到 staged Cedar CEF，或者 `libcef.so` 不包含 `cef_request_raw_snapshot`，build 会失败，而不是悄悄退回官方 CEF。

只有显式使用：

```bash
--build-arg ALLOW_OFFICIAL_CEF=1
```

才允许构建 official CEF / CDP control image。

## 为什么发布完整 distribution

理论上可以只发布一个修改后的 `libcef.so`。

v0.01 没有这么做，而是发布完整 CEF minimal distribution，主要原因是：

- `cedar-c2v-cpp` 现有 CMake 集成可以直接消费；
- headers、wrapper、CMake metadata 和 runtime resources 与 CEF 版本保持一致；
- 不需要在 consumer 里维护一套特殊的“替换 libcef.so”逻辑；
- 更容易做版本锁定和完整性检查。

也就是说，v0.01 的目标不是重新设计 CEF packaging，而是：

> 在尽量不改动 `cedar-c2v-cpp` 原有 CEF 集成方式的前提下，提供 Cedar 定制的可直接使用版本。

## Artifact identity

当前已验证 artifact：

| Item | Value |
| --- | --- |
| package size | `321,478,158` bytes |
| package SHA-256 | `7eda840f893f72764a1a14b0ad491c8d25d91507ff59c454c87ace1de9c5f2f1` |
| `libcef.so` SHA-256 | `d8bb85ad84bf5eaee62b961f6d255aaf20a7db5db8ca02a14ca684c4df4aad53` |
| custom export | `cef_request_raw_snapshot` |

v0.01 只是将已经验证过的 CEF tarball 改成 Cedar 对外 package name，文件内容不变，因此 SHA-256 不变。

如果未来重新打包、修改内容或重新编译，即使版本号相同，也必须重新计算并记录 artifact identity。

## Consumer

当前主要消费方：

- `cedar-c2v-cpp`

原则上，上层项目依赖的是：

```
Cedar version + Chromium baseline
```

即：

```
v0.01 + Chromium 152.0.7977.83
```

CEF 的完整内部版本、commit、build 参数等信息保留在本 release note 中，用于追溯和复现。
