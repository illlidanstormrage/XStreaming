# Windows 本地编译与运行

本文记录在 Windows 上从源码编译 XStreaming（Android）并本地运行的步骤，以及该项目在 Windows 下需要注意的几个坑。

## 环境要求

| 组件 | 版本 | 说明 |
| --- | --- | --- |
| Node.js | >= 20 | 本机为 v22.22.2 |
| Yarn | 4.2.2 | 通过 `corepack` 自动启用（`packageManager` 字段），不要用 yarn 1 |
| JDK | 17 | AGP 8 需要；本机为 `C:\Users\54008\.jdks\corretto-17.0.13` |
| Android SDK | platform 34 / build-tools 34.0.0 | compileSdk = 34 |
| Android NDK | 23.1.7779620 | 项目固定版本，CMake 与 Rust 交叉编译都用它 |
| CMake | 3.22.1 | 编译 `lsfg-android`（帧生成）原生库 |
| Rust | stable | 编译 `nano-rs`（原生 WebRTC 内核） |
| MinGW-w64 | 14.2.0 (ucrt) | 仅作为 Rust 在 Windows 上的**宿主编译器**（host linker） |

Rust 需要额外安装 Android 目标：

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi x86_64-linux-android
```

## 一次性准备

### 1. Android SDK 组件

```bash
sdkmanager --sdk_root=$ANDROID_HOME \
  "platform-tools" "platforms;android-34" "build-tools;34.0.0" \
  "cmake;3.22.1" "ndk;23.1.7779620" "emulator"
```

`android/local.properties` 需要指向 SDK：

```properties
sdk.dir=C\:\\Users\\<用户名>\\AppData\\Local\\Android\\Sdk
```

### 2. 安装 JS 依赖

```bash
corepack yarn install
```

> 若使用国内镜像（`YARN_NPM_REGISTRY_SERVER=https://registry.npmmirror.com`），镜像 tarball 的 checksum 与 npm 官方不一致会导致 `YN0018`，可加 `YARN_CHECKSUM_BEHAVIOR=ignore`。

### 3. Rust 宿主链接器

Windows 上 Rust 用 `x86_64-pc-windows-gnu` 工具链 + MinGW 的 `gcc` 链接宿主的 proc-macro / build script。把 MinGW 的 `bin` 和 `%USERPROFILE%\.cargo\bin` 加入 `PATH` 即可（本项目已通过用户环境变量配置好）。

crates 下载慢可在 `%USERPROFILE%\.cargo\config.toml` 加镜像：

```toml
[source.crates-io]
replace-with = "rsproxy-sparse"

[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[net]
git-fetch-with-cli = true
```

## 日常运行

需要**两个终端**。

终端 A —— 保持 Metro 常驻：

```bash
yarn start
```

终端 B —— 编译、安装、启动：

```bash
yarn android
# 等价于 react-native run-android --variant devDebug --appId com.dev.xstreaming
```

若设备/模拟器是新连上的，先建立端口反向代理（`run-android` 一般会自动做）：

```bash
adb reverse tcp:8081 tcp:8081
```

只要 APK 已经装好，之后改 JS 代码无需重新编译，Metro 热更新即可；改了 Java/C++/Rust 才需要重新 `yarn android`。

### 只出 APK

```bash
cd android
./gradlew assembleDevDebug -PreactNativeArchitectures=arm64-v8a,x86_64
# 产物: android/app/build/outputs/apk/dev/debug/app-dev-debug.apk
```

- `arm64-v8a`：真机
- `x86_64`：模拟器（x86_64 镜像）
- `armeabi-v7a`：老设备（默认配置里包含，可按需裁剪）

### 运行到模拟器

```bash
emulator -avd <AVD 名称>
adb install -r android/app/build/outputs/apk/dev/debug/app-dev-debug.apk
adb shell am start -n com.dev.xstreaming/com.xstreaming.MainActivity
```

## Windows 下的坑（已在仓库中修复）

1. **`nano-rs` 交叉编译找不到 NDK 宿主工具链**
   `android/app/build.gradle` 里 `nanoRustHostTag` 原本只判断 `darwin-arm64` / 回退 `darwin-x86_64`，Windows 上路径不存在。已改为按 `windows-x86_64`、`darwin-*`、`linux-x86_64` 顺序探测。

2. **`could not exec the linker ...-clang`**
   NDK 在 Windows 上提供的是 `aarch64-linux-android26-clang.cmd` 这类批处理包装，`llvm-ar` 只有 `.exe`。已增加 `nanoRustResolveTool`，优先取 `.exe` / `.cmd`。

3. **`patch-package` 解析失败（YN0009 / could not be parsed）**
   仓库未配 `.gitattributes`，Windows 下 `core.autocrlf=true` 会把 `patches/*.patch` 转成 CRLF，patch-package 无法解析。已把 `core.autocrlf` 关闭并把 patch 文件统一为 LF；`react-native-paper+5.12.5.patch` 的 hunk 头行数（7/7）与实际内容（6/6）不符，也一并修正为包含 `flexDirection: 'row'` 的完整 7 行上下文。

4. **`Could not download react-android-0.72.14-debug.aar … Read timed out`**
   这两个 AAR 分别是 ~200MB / ~120MB，直连 Maven Central 很容易超时。已在 `android/build.gradle` 加 `allprojects { repositories { ... } }` 优先走阿里云镜像，并在 `gradle.properties` 放宽 HTTP 超时（`systemProp.org.gradle.internal.http.socketTimeout=180000`）。

5. **模拟器没有 `libSDL2.so`**
   `third-party/sdl` 只提供了 `arm64-v8a` / `armeabi-v7a`，x86_64 模拟器上没有该库。代码里 `SdlGamepadManager` 对 `System.loadLibrary("SDL2")` 做了 try/catch 回退到 Android 输入事件桥，因此可以正常启动，只是手柄走回退实现。

## 校验点

编译成功后可确认 APK 内包含三个原生库：

```bash
unzip -l app-dev-debug.apk | grep -E "libnano_rs|liblsfg-android|libSDL2"
```

启动后 logcat 里应能看到：

```
I nano_rs native library loaded      # Rust WebRTC 内核
I SDL2 library loaded                # 仅 ARM 设备
```
