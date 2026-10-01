# cedar-render-sdk

预编译的 Cedar Render SDK 发布仓库。

本仓库**不存放实现源码**，只用于发布可直接集成和运行的构建产物，例如：

- C/C++ headers
- static libraries
- shared libraries（`.so`）
- runtime files / resources
- 版本与构建信息

## 和其它 Cedar 仓库的关系

### cedar-render-shell

Cedar Render 的实现与研发仓库。

它负责 Chromium / CEF 相关的渲染实现、实验、构建和测试。

### cedar-render-sdk

**本仓库。**

它只保存 `cedar-render-shell` 产生的预编译 SDK / release package，供其它项目直接使用，不需要重新编译 Chromium。

### cedar-c2v-cpp

上层视频生成项目。

`cedar-c2v-cpp` 可以使用这里发布的 SDK 获得 HTML/CSS/JavaScript 渲染能力，再完成后续的视频编码和输出。

简单理解：

```
cedar-render-shell
    │
    │ build
    ▼
cedar-render-sdk
    │
    │ integrate
    ▼
cedar-c2v-cpp
```

## Release package

典型的发布包会包含：

```
include/
lib/
runtime/
resources/
VERSION
```

实际目录结构以对应 Release 为准。

## 版本

SDK 版本通过 GitHub Releases 发布。

每个 Release 应尽量记录：

- SDK version
- 对应的 `cedar-render-shell` commit / tag
- Chromium / CEF version
- target platform / architecture
- build configuration
- checksum

这样可以确保上层项目能够准确复现所使用的渲染环境。

## 使用方式

从 GitHub Releases 下载需要的版本，解压后将 headers、libraries 和 runtime files 集成到目标项目中。

本仓库不接受与渲染实现相关的源码修改；相关开发工作应在 `cedar-render-shell` 中进行。
