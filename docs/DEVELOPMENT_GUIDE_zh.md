# XStreaming 开发须知

面向准备在本仓库做二次开发的人。读完后你应该知道：代码放在哪、请求怎么流动、改一个功能要动哪些文件、哪些坑不能踩。

---

## 1. 这是什么项目

开源的 Xbox / xCloud 云游戏串流客户端，Android 端（iOS 在独立分支，本仓库只有 `android/`）。

| 维度 | 选型 |
| --- | --- |
| 框架 | React Native 0.72.14 + React 18.2（**旧架构**，`newArchEnabled=false`） |
| 语言 | TypeScript（149 个 ts/tsx）+ Java（93 个）+ Rust（`nano-rs`） |
| UI | react-native-paper 5.x（MD3）+ react-navigation 6 + react-native-vector-icons |
| 状态 | redux 5（手写 reducer，**没有** redux-toolkit）+ MMKV 持久化 |
| 串流 | react-native-webrtc 124（native 引擎）/ WebView + 预构建前端包（web 引擎）/ Rust `webrtc` crate（nano 引擎） |
| 构建 | Gradle 8.0.1 + AGP（由 RN gradle 插件锁版本）、Kotlin 1.8.10、CMake、NDK 23.1.7779620 |
| 系统要求 | minSdk 26 / targetSdk 34 |
| 协议 | AGPL v3，二次开发/分发必须同样开源 |

包名分两个 flavor：`dev` → `com.dev.xstreaming`（本地开发用）、`prod` → `com.xstreaming`（上架用）。

---

## 2. 目录地图

```
src/
├── App.tsx               # 唯一入口：主题、Provider、全部路由、启动检查更新
├── i18n.ts               # i18next 初始化，10 种语言
├── Authentication.ts     # XAL 登录流程（老方案）
├── MsalAuthentication.ts # MSAL 登录流程（新方案，设置里可切换）
├── common/               # 常量与配置元数据
│   ├── settings/         # ★ 设置项声明（bases/display/gamepad/vgamepad/audio/xcloud/xhome/sensor/server/others）
│   ├── index.ts          # 按键码映射 GAMEPAD_MAPING
│   ├── usbGamepadMaping.ts
│   ├── virtualgp.ts      # 虚拟手柄布局常量
│   └── svg.ts            # 内联 SVG 字符串
├── components/           # 复用组件（VirtualGamepad / NanoStreamView / PerfPanel / SessionReportModal …）
├── pages/                # 33 个页面，一个文件一屏
├── store/                # redux store + 8 个 MMKV 持久化 store
├── tokens/               # 各阶段 token 对象（User/Sisu/Device/Xsts/Msal/Streaming）
├── xal/                  # 认证协议实现（index.ts=XAL, msal.ts=MSAL, tokenstore.ts）
├── web/                  # Xbox Web API 封装（WebApi，发主机指令）
├── xCloud/               # 云游戏 & 主机串流的会话编排（XcloudApi）
├── webrtc/               # js 侧 WebRTC 客户端 + 4 个数据通道 + 手柄驱动
├── utils/                # 日志、HTTP、截图、录制、宏、手柄导航、主题色…
├── theme/                # Paper 亮/暗主题覆盖
├── languages/            # 10 个语言包
├── assets/               # 图片（console/feedback/gamepad）
└── mock/                 # 大体积 mock 数据（data.js 900KB+, games.json 7MB）

android/app/src/main/java/com/xstreaming/   # 自研原生层
├── XalManager / XalPackage              # 登录用的设备密钥、签名、PKCE
├── FullScreenManager / PipPackage       # 全屏、画中画
├── GamepadManager / input/              # 物理手柄读取、按键包封装
├── UsbRumbleManager                     # USB / 蓝牙手柄震动
├── AudioSettingModule                   # 立体声、低延迟解码开关
├── NanoStreamView + nano/NanoRustBridge # nano 引擎的 Surface 与 JNI 桥
├── gl/VideoProcessingGLSurfaceView      # native 引擎的 GL 渲染 + FSR
├── fsr/                                 # FSR1 shader 管线
├── framegen/LsfgRuntime                 # 帧生成（需要正版 Lossless.dll）
├── touchcontrols/                       # 虚拟摇杆/按钮的原生 View
├── recording/ screenshot/               # 录屏、截图
└── webrtc/LowLatencyVideoDecoderFactory # 低延迟硬解工厂

nano-rs/src/            # Rust 原生串流内核
├── backend.rs          # AndroidSessionConfig / NanoSessionHandle（JNI 入口）
├── session.rs          # 会话状态机（Idle→Preparing→…→Connected）
├── signaling.rs transport.rs protocol.rs
├── channels/           # input / chat / control / message
├── audio_sink.rs video_sink.rs stats.rs
└── android.rs          # JNI 绑定

third-party/sdl/        # 预编译 libSDL2.so（只有 arm64-v8a / armeabi-v7a）
patches/                # patch-package 补丁，改了 node_modules 必须在这里落地
tools/                  # WebView 91 的 apk（老设备兼容测试用）
titles.json             # xCloud 标题白名单，运行时从 jsDelivr 拉取

android/app/src/main/assets/
├── stream/             # web 引擎的预构建前端（Vite 产物，index.html + 4 份 js）
└── fsr/                # FSR shader 资源
```

> `assets/stream` 是**编译产物，源码不在本仓库**。web 引擎的播放/输入逻辑要改，只能换包（或去上游 web 客户端仓库改）；本仓库能控制的是通过 `injectedJavaScriptObject` 注入的 `settings` 和 `postMessage` 消息。

---

## 3. 三个渲染引擎（最重要的心智模型）

同一套页面逻辑，三种播放实现，由设置项 `render_engine` 决定，`Home.tsx` / `Cloud.tsx` / `TitleDetail.tsx` 各自路由：

| 引擎 | 值 | 路由 | 播放实现 | 适用 |
| --- | --- | --- | --- | --- |
| WebView | `web` | `Stream` | `react-native-webview` 加载 `assets/stream/index.html`（Vite 预构建的 web 客户端），设置经 `injectedJavaScriptObject` 注入、按键经 `postMessage` 下发 | WebView ≥ 91，兼容性最好，可调 Display 参数 |
| Native | `native` | `NativeStream` / `NativePortraitStream` | `src/webrtc`（react-native-webrtc）+ `RTCFsrVideoView`（GL + FSR） | 低延迟、支持 FSR / 帧生成 |
| Nano | `nano` | `NanoStream` | `nano-rs`（Rust webrtc）+ `NanoStreamView`（原生 Surface） | 新引擎，最低开销 |

要点：

- **`src/webrtc` 只服务 native 引擎**：4 条数据通道、手柄驱动、包格式都在这里；web 引擎的同类逻辑在预构建包里，JS 侧只负责消息桥（`Stream.tsx` 的 `handleWebviewMessage` / `postWebviewMessage`）。
- **改公共行为要三处对齐**：手柄映射、统计上报、震动这类跨引擎逻辑，`Stream.tsx`（消息桥）、`NativeStream.tsx`（JS 通道）、`NanoStream.tsx`（Rust 通道）都要过一遍（三者各 1.3k～3.7k 行）。
- 老设备兜底：`Home.tsx` 会检测 WebView < 91 且 Android < 12 时强制走 native。
- `web` 引擎下 "显示"（Display）设置页才有意义，native/nano 下点进去会提示不支持。
- x86_64（模拟器）没有 `libSDL2.so`，SDL 手柄库加载失败会回退到 Android 输入桥，不影响启动。

### 串流链路（按调用顺序）

```
Login.tsx
  └─ Authentication / MsalAuthentication
       └─ xal/*  换取 UserToken → SisuToken → DeviceToken → XstsToken
            ├─ webTokenStore      (Web API 用)
            └─ streamingTokenStore(串流用，带过期判断)

Home.tsx / Cloud.tsx / TitleDetail.tsx
  └─ XcloudApi.startSession()       // 云游戏 vs 主机，两条分支
       → waitState()                // 轮询会话状态
       → 取 configuration / 交换 SDP offer & answer / 交换 ICE candidate
       → 建立 4 条 DataChannel: input, chat, control, message
       → 播放：web / native / nano 三选一
       → webrtc/Packet 打包按键 → InputChannel.send()
```

协议细节在 `docs/1.Auth_zh.md`、`2.Web_zh.md`、`3.Stream_zh.md`，改协议前务必先读。

---

## 4. 状态管理：两个体系，别混用

**redux（内存态，进程结束即丢）** — `src/store/index.ts`，只有 7 个字段：

```ts
isLogined, streamingTokens, webToken, authentication, redirect, profile, stars
```

**MMKV（持久化）** — `src/store/*Store.ts`，每个都是"模块级缓存 + JSON 序列化"的简单封装：

| 文件 | key | 内容 |
| --- | --- | --- |
| `settingStore.ts` | `user.settings` | 全部用户设置，含 57 个设置项 |
| `xcloudStore.ts` | `user.xcloud` | 云游戏库缓存（标题列表/收藏/缓存时间） |
| `consolesStore.ts` | — | 主机列表缓存 |
| `gamepadStore.ts` | — | 自定义按键映射、虚拟手柄布局 |
| `streamTokenStore.ts` / `webTokenStore.ts` | — | 两类 token 与有效期 |
| `shortcutStore.ts` | — | 桌面快捷方式对应标题 |

约定：

- 组件里**同步**读 `getSettings()`（命中模块级 `cachedSettings`），写完调 `saveSettings()`。
- `saveSettings` 会做三件事：写 MMKV、更新缓存、`DeviceEventEmitter.emit('SETTINGS_CHANGED')` 通知 `App.tsx` 重渲染。**不要自己另开一套 settings 状态**。
- 部分设置会顺带同步给原生（如 `AudioSettingModule.setStereoEnabled`），新增这类项时在 `saveSettings` 里补。

### 新增一个设置项的完整清单

设置项是"声明式"的，元数据与读写逻辑分离。

1. **声明**：在 `src/common/settings/<分组>.ts` 的数组里加一个对象，`name` 必须与 `Settings` 字段同名。支持 `type`：`radio` / `color` / `slider` / `action`（`action` 用于触发一个操作，如导入 DLL）。
2. **类型与默认值**：`src/store/settingStore.ts` 里补 `Settings` 类型字段 + `defaultSettings` 默认值（漏了默认值会导致 `undefined` 传进原生）。
3. **文案**：`src/languages/*.ts` 十个语言包都加 key。**key 就是英文原句**（如 `t('Settings')`），新增语言容易漏，建议以 `en.ts` 为准同步。
4. 页面渲染是自动的：`Settings.tsx` 遍历各分组 `forEach` 生成列表，`SettingDetail.tsx` 汇总所有分组按 `name` 找元数据。**只有新增分组时**才需要改这两个页面。

注意 `Settings.tsx` 用手柄/遥控器上下导航，（列表索引是手工 push 顺序维护的，插入自定义行时记得在 `settingActions` 同位置补一个 `list.push`，否则焦点错位）。

---

## 5. 原生层怎么加东西

### 5.1 新增一个 Java 原生模块（3 步）

```java
// 1) Module
public class FooModule extends ReactContextBaseJavaModule {
  public FooModule(ReactApplicationContext ctx) { super(ctx); }
  @Override public String getName() { return "FooModule"; }   // JS 里 NativeModules.FooModule
  @ReactMethod public void doSomething(String arg, Promise promise) {
    try { promise.resolve("ok"); } catch (Exception e) { promise.reject(e); }
  }
}

// 2) Package
public class FooPackage implements ReactPackage {
  @Override public List<NativeModule> createNativeModules(ReactApplicationContext ctx) {
    List<NativeModule> modules = new ArrayList<>();
    modules.add(new FooModule(ctx));
    return modules;
  }
  @Override public List<ViewManager> createViewManagers(ReactApplicationContext ctx) {
    return Collections.emptyList();
  }
}

// 3) MainApplication.getPackages() 里手动 add（本项目不用 autolink 注册自研模块）
packages.add(new FooPackage());
```

自定义 View 走 `ViewManager` + `requireNativeComponent`（参考 `NanoStreamView.tsx` + `NanoStreamViewManager`）。

### 5.2 已有原生能力（避免重复造）

Module：`XalManager`、`FullScreenManager`、`GamepadManager`、`UsbRumbleManager`、`BatteryModule`、`GamepadSensorModule`、`AudioSettingModule`、`UpdateManager`、`NativeInputDialog`、`ConfigTransferModule`、`FrameGenAssetModule`、`ShortcutManager`、`GameRecorder`、`Screenshot`、`SensorPackage`。
ViewManager：`AnalogStick`、`ButtonView`、`RTCFsrVideoView`、`NanoStream`（Surface）。
Service：`UsbDriverService`（USB 手柄权限）。

### 5.3 Rust（nano-rs）

- crate 名 `nano-rs`/`nano_rs`，`crate-type = ["rlib", "cdylib"]`，产物 `libnano_rs.so`。
- 构建由 Gradle 任务 `syncNanoRustJniLibs` 驱动（挂在 `preBuild` 上）：按 `reactNativeArchitectures` 逐 ABI 调 `cargo build --target <triple> --release`，把 `.so` 拷进 `app/build/generated/nanoJniLibs/<abi>/`。输入文件没变时 Gradle 会跳过，第一次编译 webrtc 依赖约 3～6 分钟，之后增量。
- ABI ↔ triple：`armeabi-v7a → armv7-linux-androideabi`、`arm64-v8a → aarch64-linux-android`、`x86_64 → x86_64-linux-android`。
- JNI 入口 `backend.rs`（`AndroidSessionConfig` / `NanoSessionHandle::start/send_gamepad_frame/stop`），Java 侧 `nano/NanoRustBridge.java`。库加载失败被 try/catch 兜住（`NATIVE_AVAILABLE=false`），所以 nano 不可用时 App 仍能启动，只是不能走 nano 引擎 —— 排查时**不要只看"没崩"就以为加载成功**。
- 依赖 `webrtc 0.17.2` / `tokio` / `reqwest(rustls)`，改 Cargo 依赖后注意 vendored 编译会明显变慢。

---

## 6. 代码风格与约定

Prettier（`.prettierrc.js`）：单引号、**对象花括号内不留空格** `{a, b}`、`arrowParens: 'avoid'`（`x => x`）、`trailingComma: 'all'`、`bracketSameLine: true`。

ESLint 继承 `@react-native`。TypeScript 放宽了 `noImplicitAny`/`useUnknownInCatchVariables`，但**不要**因此到处 `any`，`store`/`protocol`/`native bridge` 已有类型可复用。

其他约定：

- **日志**统一 `const log = debugFactory('模块名')`（`utils/debug.ts`，底层 react-native-logs），不要裸 `console.log`。
- **i18n**：所有面向用户的字符串走 `t()`，key 用英文原句；新增文案要补 10 个语言包。
- **主题色**：用 `useTheme()` 取 Paper 主题色；需要硬编码色值时用 `utils/themeColor.ts` 的 `normalizeHexColor` / `applyPrimaryColorToPaperTheme`，跟随用户自定义主色。
- **页面注册**：`App.tsx` 里 import → 用 `withPageBackground`（带 Xbox 水印背景）或 `withSolidPageBackground` 包一层 → 添加到 `RootStack.Screen`；模态页放进 `presentation: 'modal'` 那个 Group。
- **手柄/电视可用性**：可导航页面用 `useGamepadNavigation`，并在 `Platform.isTV` 下校验焦点顺序；新页面别只考虑触屏。
- **改了 `node_modules`**：必须 `npx patch-package <包名>` 生成 `patches/*.patch`。现有 5 个补丁都改了运行时行为（spinkit 动画、spinner-overlay 类型与参数、paper 分段按钮内边距、webrtc 视频 View、webview 的 view 管理器），删掉或丢失会导致行为回归，动它们之前先读 patch 内容。
- **`patches/*.patch` 必须保持 LF**。仓库已设 `core.autocrlf=false`，不要改回 true，否则 patch-package 会解析失败（Windows 上踩过，见 `Local_Build_Windows_zh.md`）。

---

## 7. 本地开发流程

环境搭建见 [`docs/Local_Build_Windows_zh.md`](./Local_Build_Windows_zh.md)。日常两条命令：

```bash
yarn start      # 终端 A：Metro 常驻
yarn android    # 终端 B：编译 devDebug + 安装 + 启动
```

只改 JS 不用重编，Metro 热更新；改了 Java / C++ / Rust 必须重跑 `yarn android`。

### 调试手段

| 目的 | 手段 |
| --- | --- |
| 看业务日志 | `adb logcat -s ReactNativeJS`，日志带 namespace |
| 看原生/解码日志 | `adb logcat` 后过滤 `nano_rs` / `lsfg` / `SDL2` / `webrtc` 关键字 |
| 调试页 | 设置 → 进入调试（token 打印、震动、音频轨、WebView 调试） |
| 性能面板 | 设置里开 `show_performance`，串流页显示帧率/码率/丢包 |
| 会话报告 | `SessionReportModal`（`show_session_report`） |
| 改配置不想重装 | 设置 → 配置迁移，导出/导入 JSON |

### 提交前自查

```bash
yarn lint
npx tsc --noEmit      # 无 npm script，直接跑 tsc
yarn test             # 仅 2 个测试文件，别指望覆盖率
```

- 不要提交 `android/local.properties`、keystore、`*.log`。
- `gradle.properties` 里带着 release 签名口令（历史遗留），改动时不要把真实密钥提交上来。
- `nano-rs/target/`、`android/app/build/`、`android/app/.cxx/`、`.gradle/` 都是构建产物。

---

## 8. "我想改 X，该看哪里"

| 需求 | 主要文件 |
| --- | --- |
| 首页主机列表 | `pages/Home.tsx`、`components/ConsoleItem.tsx`、`store/consolesStore.ts` |
| 云游戏库（最大文件，4.5k 行） | `pages/Cloud.tsx`、`xCloud/index.ts`、`xCloud/ownedLibrary.ts` |
| 登录流程 | `pages/Login.tsx`、`Authentication.ts`、`MsalAuthentication.ts`、`xal/*`、`tokens/*` |
| 串流画面（三选一） | `pages/Stream.tsx` / `pages/NativeStream.tsx` / `pages/NanoStream.tsx` |
| web 引擎前端（预构建包） | `android/app/src/main/assets/stream/`（无源码，只能整体替换） |
| 画面处理（FSR） | `gl/VideoProcessingGLSurfaceView.java`、`fsr/*`、`components/RTCFsrView.tsx` |
| 帧生成（LSFG） | `framegen/LsfgRuntime.java`、`cpp/lsfg_vendor/*`、`FrameGenAssetModule`、设置项 `smart_frame_generation` |
| 物理手柄/震动 | `GamepadManager.java`、`UsbRumbleManager`、`input/*`、`common/usbGamepadMaping.ts` |
| 虚拟手柄 | `components/VirtualGamepad*.tsx`、`touchcontrols/*`、`common/virtualgp.ts`、`store/gamepadStore.ts` |
| 宏 | `utils/virtualMacro.ts`、`pages/VirtualMacroSettings.tsx` |
| 设置页 | `common/settings/*`、`store/settingStore.ts`、`pages/Settings.tsx`、`pages/SettingDetail.tsx` |
| 多语言 | `languages/*.ts`、`utils/locale.ts` |
| 主题 | `theme/index.ts`、`theme/index.dark.ts`、`utils/themeColor.ts` |
| 更新检查/自升级 | `utils/updater.ts`、`UpdateManager.java` |
| 录屏/截图 | `recording/*`、`screenshot/*`、`utils/gameRecorder.ts`、`utils/screenshot.ts` |

---

## 9. 常见坑（按踩坑频率排序）

1. **红屏 "Unable to load script"** —— Metro 没跑或 `adb reverse` 丢了。先 `yarn start`，再 `adb reverse tcp:8081 tcp:8081`。
2. **改了 `node_modules` 但在别人机器上不生效** —— 忘了 `patch-package`。
3. **改完 Rust 行为没变化** —— `syncNanoRustJniLibs` 的 `inputs` 只匹配 `Cargo.toml/Cargo.lock/build.rs/src/**/*.rs`，改了别的文件（比如新增子目录外的文件）要 `./gradlew --rerun-tasks :app:syncNanoRustJniLibs`，或直接删 `nano-rs/target`。
4. **模拟器能跑、真机白屏（或反过来）** —— ABI 不匹配。`reactNativeArchitectures` 默认只有 `armeabi-v7a,arm64-v8a`，模拟器要显式加 `x86_64`。
5. **设置项改了没生效** —— 检查是不是漏了 `defaultSettings`，或者当前渲染引擎下该设置本来就不生效（Display 系列只在 `web`）。
6. **音量/震动/立体声不对** —— `saveSettings` 里同步给原生的那几个调用被绕过了（直接改 MMKV 就会这样）。
7. **`git pull` 后所有文件显示被修改** —— 换行符被 `core.autocrlf=true` 改写，改回 `false` 再 `git checkout -- .`。
8. **帧生成没有效果** —— 需要正版 `Lossless.dll`（项目不提供），并且必须走 native 引擎 + 设备支持 Vulkan。

---

## 10. 上手路径建议

1. 先跑通 `yarn start` + `yarn android`，能进登录页。
2. 从**设置页**入手（纯 JS、有类型、有现成模板）：加一个开关，体会 i18n + store + 页面三层怎么串。
3. 再动**一个页面**（如 `History.tsx` / `Search.tsx` 这类小页），熟悉 `withPageBackground` 与导航参数。
4. 然后才碰**串流三兄弟**：先把 `src/webrtc` 的通道与 `webrtc/Packet` 读懂，再看三个 `*Stream.tsx` 的差异。
5. 原生/Rust 改动前，先用 `adb logcat` 确认现有链路在跑（`nano_rs native library loaded` / `SDL2 library loaded`），避免把"本来就没启用"误判成"我改坏了"。
