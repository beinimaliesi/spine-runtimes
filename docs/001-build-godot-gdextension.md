# 构建 Godot GDExtension

本文说明本仓库 Spine 4.3 runtime 的 GDExtension 构建方式。构建产物是供 Godot 加载的动态库，无需重新编译 Godot 编辑器。

## 1. Build on Local

### 版本信息

| 组件 | 版本 | 说明 |
| --- | --- | --- |
| Spine runtime | `4.3` | 当前仓库运行时 |
| Godot | `4.7.2-stable` | 编辑器与导出模板使用相同版本 |
| godot-cpp | `master` | 为保证构建可复现，应固定并记录使用的 commit |
| godot-cpp API | `4.7` | SCons 参数 `api_version=4.7` |
| Emscripten | `4.0.11` | Web 构建采用 Godot 4.7.2 官方 Web 工作流指定的版本 |
| SCons | `4.11.1` | 构建工具版本 |

Emscripten 版本依据：[Godot 4.7.2 Web 构建配置](https://raw.githubusercontent.com/godotengine/godot/4.7.2-stable/.github/workflows/web_builds.yml) 中的 `EM_VERSION: 4.0.11`。

Web 构建前先加载 emsdk 环境，并用 `emcc --version` 确认版本为 `4.0.11`，避免误用 PATH 中的其他版本。当前仓库 CI 指定的版本为 `4.0.20`，与上述官方配置不同。

当前 godot-cpp Web 构建仅支持 wasm32，产物需配合相同架构和线程模式的 Godot Web 导出模板。上述官方工作流的 threaded 构建配置为 wasm64；集成时应核对模板架构，不能仅凭 Emscripten 版本一致判断兼容性。

在已有 emsdk 目录中安装并切换版本：

```bash
./emsdk install 4.0.11
./emsdk activate 4.0.11
source ./emsdk_env.sh
emcc --version
```

然后在同一终端进入本仓库执行 Web 构建命令。若 emsdk 检测到系统 Python 版本过低，可通过 `EMSDK_PYTHON` 指定满足要求的 Python 路径。

### 本地准备

基础工具为 Git、Python 3、SCons 和目标平台的 C++ 工具链。macOS 需要可用的 Xcode 命令行工具；iOS 需要完整 Xcode 和 iOS SDK。

首次初始化：

```bash
cd spine-godot/build
bash setup-extension.sh master false
cd ../..
```

该脚本会克隆 `godot-cpp/master`，复制 `spine-cpp` 到 `spine-godot/spine_godot/spine-cpp`，并准备示例工程的扩展配置。`false` 表示不下载和编译 Godot 编辑器。

**不要在日常增量编译前重复执行 setup。** 它会删除并重建 `spine-godot/godot-cpp`、`spine-godot/spine_godot/spine-cpp` 和 `spine-godot/example-v4-extension/addons/godot_lottie/bin`。

### macOS 本地开发

在 Apple Silicon 机器上，可显式指定 `arm64`，编译 editor、debug 和 release 三种 framework：

```bash
cd spine-godot
for target in editor template_debug template_release; do
  scons platform=macos arch=arm64 target="$target" \
    api_version=4.7 generate_bindings=yes \
    -j"$(sysctl -n hw.logicalcpu)"
done
cd ..
```

需要同时支持 Intel 和 Apple Silicon 时，将 `arch=arm64` 改为 `arch=universal`。

也可以使用仓库脚本，架构采用所克隆 godot-cpp 的默认配置：

```bash
GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh macos
```

注意：当前 `build-extension.sh` 仅对 Linux 和 Android 转发第二个架构参数；`build-extension.sh macos arm64` 不会将 `arm64` 传给 SCons。需要明确架构时使用上面的直接编译命令。

产物自动复制到：

```text
spine-godot/example-v4-extension/addons/godot_lottie/
├── spine_godot_extension.gdextension
└── bin/
    └── macos/
        ├── libspine_godot.macos.editor.framework
        ├── libspine_godot.macos.template_debug.framework
        └── libspine_godot.macos.template_release.framework
```

后续修改 `spine-godot/spine_godot` 下的 C++ 源码，重新执行对应 SCons 命令即可增量编译。只验证编辑器时可仅编译 `target=editor`；运行导出的 Debug/Release 应用需对应的 template 库。若修改了根目录的 `spine-cpp`，还需先同步到构建使用的 `spine-godot/spine_godot/spine-cpp` 副本。

### 其他平台

以下命令均在完成初始化后执行。交叉编译还需目标平台工具链；Windows 和 Linux 可优先使用对应系统或 GitHub Actions。

| 平台 | 工具链要求 | 构建命令 |
| --- | --- | --- |
| iOS | Xcode、iOS SDK | `GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh ios` |
| Android arm64 | Android SDK、NDK | `GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh android arm64` |
| Android x86_64 | Android SDK、NDK | `GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh android x86_64` |
| Web | Emscripten，`emcc` 在 PATH 中 | `GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh web` |
| Linux x86_64 | Linux C++ 工具链 | `GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh linux x86_64` |
| Windows | Windows C++ 工具链 | `GODOT_CPP_API_VERSION=4.7 bash spine-godot/build/build-extension.sh windows` |

当前 CI 安装 Android NDK `28.1.13356709`，并通过 `ANDROID_NDK_ROOT` 传入路径。构建时通过 `ANDROID_NDK_ROOT` 指定对应 NDK 的安装路径。例如：

```bash
ANDROID_NDK_ROOT="$ANDROID_HOME/ndk/28.1.13356709" \
GODOT_CPP_API_VERSION=4.7 \
bash spine-godot/build/build-extension.sh android arm64
```

平台产物位于 `spine-godot/example-v4-extension/addons/godot_lottie/bin/<平台>/`。桌面平台构建 editor/debug/release；iOS、Android 构建 debug/release；Web 额外构建 threads 和 nothreads 两种变体。

### 加载验证与集成

使用匹配版本的 Godot 编辑器加载示例工程。以下以 macOS 默认应用路径为例：

```bash
/Applications/Godot.app/Contents/MacOS/Godot --version

/Applications/Godot.app/Contents/MacOS/Godot \
  --headless --path spine-godot/example-v4-extension \
  --editor --import
```

检查退出码以及日志中的 GDExtension 加载错误、缺失符号和脚本错误。headless 加载只能验证导入和加载，不能验证批处理后的画面；还需打开示例场景检查颜色、裁剪和 SlotNode 遮挡顺序。

集成到业务工程时，将整个 `addons/godot_lottie` 目录复制到项目的 `res://addons/godot_lottie`，可只保留需要的平台产物。例如：

```text
业务工程/addons/godot_lottie/
├── spine_godot_extension.gdextension
└── bin/
    ├── android/
    ├── ios/
    ├── web/
    ├── macos/
    │   └── libspine_godot.macos.*.framework
    └── windows/
```

`.gdextension` 中的库路径固定为 `res://addons/godot_lottie/bin/<平台>/...`，因此必须安装到项目的 `addons/godot_lottie` 目录。Godot 会自动发现并加载 GDExtension，无需额外添加 `plugin.cfg`。保持目录结构不变，并确保库的架构与运行 Godot 的进程一致。Linux 产物同样放在 `bin/linux/`。

### 参考

- [本仓库 GDExtension SConstruct](../spine-godot/SConstruct)
- [依赖初始化脚本](../spine-godot/build/setup-extension.sh)
- [多平台构建脚本](../spine-godot/build/build-extension.sh)
- [Spine 官方 GDExtension 编译文档](https://zh.esotericsoftware.com/spine-godot#编译-GDExtension)

## 2. Build via GitHub Actions

### 版本配置

本文使用的构建配置为：

| Godot tag | godot-cpp API |
| --- | --- |
| `4.7.2-stable` | `4.7` |

[GitHub Actions 构建矩阵](../.github/workflows/spine-godot-extension-v4-all.yml) 当前仅启用 `4.7.2-stable`。

### 触发构建与下载产物

多平台构建可直接使用仓库现有工作流：

1. 将需要构建的代码推送到 GitHub 分支。
2. 打开仓库的 **Actions** 页面，选择 **Build spine-godot GDExtension (All Godot 4.x versions)**。
3. 点击 **Run workflow**，选择目标分支并启动。
4. 完成后，在该次运行的 **Artifacts** 中下载对应平台的 `4.7.2-stable` 产物。

入口为 [spine-godot-extension-v4-all.yml](../.github/workflows/spine-godot-extension-v4-all.yml)，具体构建步骤在 [spine-godot-extension-v4.yml](../.github/workflows/spine-godot-extension-v4.yml)。当前会构建 Windows、Linux x86_64、macOS、iOS、Android 和 Web；Artifacts 保留时间配置为 1 天。

单平台 Artifact 主要包含编译产物：解压到 `res://addons/godot_lottie/bin/<平台>/`，并将仓库中的 `spine-godot/spine_godot_extension.gdextension` 复制到 `res://addons/godot_lottie/`。工作流的 S3 汇总压缩包则直接包含 `addons/godot_lottie/` 目录结构。

下载后，按第一章的“加载验证与集成”步骤检查扩展加载和画面效果。
