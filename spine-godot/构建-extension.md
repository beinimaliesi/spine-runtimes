# spine-godot GDExtension 构建

本文说明 Spine 4.3 runtime 面向 Godot 4.7.2 的 GDExtension 构建流程。

## 版本和工具链

- Godot：`4.7.2-stable`
- godot-cpp：`master`
- godot-cpp API：`4.7`
- macOS：Xcode、SCons
- Android：Android SDK、NDK
- Web：Emscripten
- Linux / Windows：建议使用 `linux/amd64` Docker 容器

不要使用 godot-cpp 的 `4.3` 分支构建当前源码。当前 `SpineSprite3D.cpp` 使用的 Godot API 在该分支中不存在。

## 初始化 godot-cpp

在 `spine-godot/build` 目录执行：

```bash
./setup-extension.sh master false
```

脚本会在 `spine-godot/godot-cpp` 克隆 master，并准备 `spine_godot/spine-cpp` 和示例工程的 `bin` 目录。

## macOS

构建 editor、debug 和 release 三套 universal framework：

```bash
cd spine-godot/build
GODOT_CPP_API_VERSION=4.7 ./build-extension.sh macos
```

产物位于：

```text
spine-godot/example-v4-extension/bin/macos/
```

## iOS

需要 Xcode 和 iOS SDK：

```bash
cd spine-godot/build
GODOT_CPP_API_VERSION=4.7 ./build-extension.sh ios
```

该命令生成 arm64 的 debug/release framework：

```text
spine-godot/example-v4-extension/bin/ios/
```

## Android

`build-extension.sh` 默认查找 NDK `28.1.13356709`。如果实际安装的是其他版本，可以建立临时 SDK 映射：

```bash
mkdir -p /tmp/spine-godot-android-sdk/ndk
ln -s "$ANDROID_HOME/ndk/28.2.13676358" \
  /tmp/spine-godot-android-sdk/ndk/28.1.13356709
```

构建 arm64：

```bash
cd spine-godot/build
ANDROID_HOME=/tmp/spine-godot-android-sdk \
GODOT_CPP_API_VERSION=4.7 \
./build-extension.sh android
```

构建 x86_64：

```bash
ANDROID_HOME=/tmp/spine-godot-android-sdk \
GODOT_CPP_API_VERSION=4.7 \
./build-extension.sh android x86_64
```

产物位于 `spine-godot/example-v4-extension/bin/android/`，包括两种架构的 debug/release `.so`。

## Web

需要 `emcc` 已加入 `PATH`。构建 threaded 和 nothreads 的 debug/release 变体：

```bash
cd spine-godot/build
GODOT_CPP_API_VERSION=4.7 ./build-extension.sh web
```

产物位于 `spine-godot/example-v4-extension/bin/web/`：

```text
libspine_godot.web.template_debug.wasm32.wasm
libspine_godot.web.template_release.wasm32.wasm
libspine_godot.web.template_debug.wasm32.nothreads.wasm
libspine_godot.web.template_release.wasm32.nothreads.wasm
```

Emscripten 可能输出 `WASM_BIGINT is deprecated`，该提示不影响构建结果。

## Linux

TODO：Linux 构建暂不支持，后续补充 Linux 交叉编译工具链和验证流程。

## Windows

macOS 默认没有 Windows MinGW 工具链，建议使用 amd64 Docker 容器：

```bash
docker run --rm --platform=linux/amd64 \
  -v /Users/tuxianchao/ws/workspace-godot-engine/spine-runtimes:/src \
  -w /src/spine-godot/build \
  ubuntu:24.04 bash -lc '
    set -e
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y --no-install-recommends \
      build-essential python3 python3-pip pkg-config mingw-w64
    python3 -m pip install --break-system-packages scons

    MINGW_PREFIX=/usr GODOT_CPP_API_VERSION=4.7 ./build-extension.sh windows
  '
```

Windows 产物位于 `example-v4-extension/bin/windows/`，默认生成 x86_64 的 editor/debug/release 变体。

如果 Docker Hub 或 Debian 包源不可访问，容器无法安装 MinGW；需要先修复 Docker 网络或改用已准备好工具链的构建机。

## 验证

检查产物格式：

```bash
file spine-godot/example-v4-extension/bin/macos/*/libspine_godot.* \
  spine-godot/example-v4-extension/bin/ios/*/libspine_godot.* \
  spine-godot/example-v4-extension/bin/android/*.so \
  spine-godot/example-v4-extension/bin/web/*.wasm
```

使用 Godot 4.7.2 headless 加载示例工程：

```bash
/Applications/Godot.app/Contents/MacOS/Godot \
  --headless \
  --path spine-godot/example-v4-extension \
  --editor \
  --quit
```

验证业务工程主场景：

```bash
/Applications/Godot.app/Contents/MacOS/Godot \
  --headless \
  --path /Users/tuxianchao/ws/workspace-gd-level-game \
  --quit-after 2
```

成功标准是进程退出码为 `0`，且日志中没有 GDExtension 加载错误。构建输出默认被 `.gitignore` 忽略；复制到其他工程时，需要同时复制 `spine_godot_extension.gdextension` 和对应平台目录。
