# Code Client — 远端 VS Code Web 的 Android 客户端

- 日期：2026-09-23
- 状态：待评审
- 仓库：https://github.com/leoyyong/code-client-app （分支 `master`）
- 许可：MIT

## 1. 目标

一个 Android 应用，启动时提供地址输入框；确认后以**纯页面**全屏显示远端 VS Code Web 端，不显示标题栏、地址栏或任何应用自有 UI。

必须支持三种目标端点：

1. 自建 code-server，有域名 + HTTPS（证书可能自签名）
2. 局域网 IP + 端口，可能走明文 HTTP
3. `vscode.dev` / `github.dev` 官方端点

## 2. 定位

用 Rust 编写的 Android 应用，通过 JNI 驱动系统 `android.webkit.WebView` 加载远端 VS Code Web 界面。

## 3. 能力边界（先说清楚做不到什么）

这三条是平台硬约束，不是实现取舍。列为需求的一部分，避免验收时产生误解。

| 边界 | 说明 |
|---|---|
| **"打开本地文件夹"不可用** | VS Code Web 的 File ▸ Open Folder 依赖浏览器 File System Access API（`window.showDirectoryPicker`），该 API 在 Android WebView 中不存在（Chrome for Android 同样不支持）。**访问远端 code-server 的工作区不受影响**，因为文件本来就在服务器上。真正可用的能力是"从手机上传文件到远端工作区"。 |
| **Rust 无法独立驱动 WebView** | 文件上传需要继承 `WebChromeClient`、证书放行需要继承 `WebViewClient`。JNI 不能继承 Java 类，`java.lang.reflect.Proxy` 只能代理接口。因此必须存在一层 Kotlin。 |
| **长按返回不能靠 `KEYCODE_BACK`** | 开启 predictive back 后旧 `KEYCODE_BACK` 事件不再触发。必须用 AndroidX `OnBackPressedCallback` + 延时判定。Firefox Android 为同样需求采用此法（[Mozilla D234653 / Bug 1932300](https://phabricator.services.mozilla.com/D234653?id=973301)）。 |

## 4. 已确认的决策

| 议题 | 决定 | 理由 |
|---|---|---|
| 技术路线 | Rust 主体 + 薄层 Kotlin 平台胶水 | "零 Kotlin" 无法满足文件上传与证书放行 |
| 入口 | 普通 `AppCompatActivity` + Rust `cdylib` 经 JNI 加载 | 不用 NativeActivity / `android-activity` / `cargo-apk`；换取正常生命周期、`onActivityResult`、AndroidX |
| 构建 | `cargo-ndk` + Gradle，不使用 `rust-android-gradle` 插件 | 少一层魔法，CI 每步可见 |
| CI | GitHub Actions，产出可侧载的 debug APK | 可重复、不污染本机 |
| 配置持久化 | Rust 用 `std::fs` 写 app 私有目录 | 状态留在 Rust |
| 输入界面 | 原生 `EditText` 对话框 | 让 code-server 页面在输入框弹出时保持存活，不触发重连 |
| 地址记忆 | 记住上次地址，冷启动直接进入 | 日常免手动输入 |
| 回输入框 | 长按返回（≥500 ms） | 配合三级返回链构成逃生舱口 |
| 短按返回 | 网页后退 → 回输入框 → 退出 | 符合直觉 |
| 沉浸式 | 隐藏状态栏，保留导航手势条 | 页面面积最大且不会困住用户 |
| 证书策略 | 默认询问，可按主机记住允许 | 兼顾自签名可用性与公共网络安全 |
| minSdk | 29（Android 10） | 免掉全部存储权限代码 |
| targetSdk / compileSdk | 35 | 当前稳定基线 |
| 应用 ID | `io.github.leoyyong.codeclient` | 与 GitHub 托管项目对应 |
| Rust 库名 | `codeclient` → `libcodeclient.so` | |
| 启用适配 | 剪贴板、软键盘不遮挡、后台保活、横屏、文件上传、文件下载、外链跳系统浏览器 | 用户全选 |
| 地址规范化 | 无 scheme 时：IP 字面量补 `http://`，域名补 `https://` | 局域网 code-server 通常明文，公网通常 HTTPS |

## 5. 架构

### 5.1 分层

```
APK
├─ Kotlin 层  (~180 行 / 7 个文件)  ← 仅平台强制项，全是薄转发，无判断逻辑
│   MainActivity          AppCompatActivity；生命周期与返回键转发；UI 线程跳板
│   AppWebViewClient      继承 WebViewClient   → 转发 SSL / 导航 / 页面 / 错误 / 渲染进程崩溃
│   AppWebChromeClient    继承 WebChromeClient → 转发文件选择 / 权限请求 / 进度
│   AppDownloadListener   实现 DownloadListener（接口）→ 转发下载
│   InputDialog           地址输入对话框
│   SslPrompt             证书确认对话框
│   FileChooser           ActivityResultContracts 文件选择管道
│
└─ Rust 层  (libcodeclient.so, cdylib)  ← 应用本体，所有决策在此
    lib.rs         JNI 导出入口
    jni_env.rs     JNI 封装：GlobalRef、局部引用、异常检查、JString↔String
    app.rs         应用状态：Activity/WebView 全局引用、当前 URL、导航状态
    webview.rs     WebView 创建与操作
    nav.rs         ← 返回键状态机          纯逻辑，可单测
    url.rs         ← 地址校验与规范化      纯逻辑，可单测
    ssl.rs         ← 按主机允许列表        纯逻辑，可单测
    config.rs      ← 配置读写（std::fs）   纯逻辑，可单测
    ui.rs          沉浸式、屏幕方向
    downloads.rs   DownloadManager 调用
    intents.rs     外链 ACTION_VIEW
```

### 5.2 边界划分原则

Kotlin 只保留三类**平台强制**的东西：继承 Java 类、实现 Java 接口、启动 Android UI 管道。每一处都不含判断逻辑。所有**决策**（是否放行证书、返回键该退网页还是退应用、地址是否合法、下载存哪、外链是否跳出）都在 Rust。

由此得到两个可验证的性质：

- Kotlin 层可整层替换而不影响 Rust
- `nav` / `url` / `ssl` / `config` 四个模块无 Android 依赖，可在宿主机单测

### 5.3 数据流

事件方向（Kotlin → Rust）：Android 回调发生 → Kotlin 薄转发 → JNI → Rust 决策。

命令方向（Rust → Android）：Rust 通过 JNI 调用 Android API（`loadUrl` / `setContentView` / `DownloadManager` / `startActivity` / 对话框）。

Kotlin 不知道"为什么"，只知道"转发去哪"。

### 5.4 线程模型

`WebView` 只能在 UI 线程被触碰。分三种情形：

1. Kotlin 回调本身已在 UI 线程 → Rust 在其内部同步调用 JNI，安全。
2. Rust 后台线程（仅加载超时计时器）需要操作 UI → 调用 `MainActivity.postToUi()`，该方法内部 `runOnUiThread(uiTrampoline)`；`uiTrampoline` 是 Kotlin 里预先构造的单个 `Runnable`，回调 `nativeOnUiTrampoline()`，Rust 在该回调里执行实际操作。
   - 只预构造一个 `Runnable` 是因为 Rust 无法便捷地创建 Java 对象；这把跨线程编组压缩成一次无参调用。
3. 所有 Rust→Android 的调用都以 `Activity::postToUi` 或已确定的 UI 线程上下文为前提。

## 6. JNI 接口

### 6.1 Kotlin 声明的 `external fun`（Rust 实现）

```
MainActivity
  nativeCreate(activity: Activity, filesDir: String)
  nativeOnResume() / nativeOnPause() / nativeOnDestroy()
  nativeOnBackPressed(): Boolean          返回 true 表示已消费
  nativeOnBackLongPress()
  nativeOnUiTrampoline()
  nativeOnUrlSubmitted(url: String)       输入对话框确认
  nativeOnInputDialogVisibility(visible: Boolean)
  nativeOnSslChoice(allow: Boolean, remember: Boolean)

AppWebViewClient
  nativeOnPageStarted(url: String)
  nativeOnPageFinished(url: String)
  nativeOnReceivedError(code: Int, description: String, url: String, isMainFrame: Boolean)
  nativeOnReceivedHttpError(statusCode: Int, url: String, isMainFrame: Boolean)
  nativeUrlPolicy(url: String, hasGesture: Boolean): Int
                                          0=内部加载 1=跳系统浏览器 2=已由 Rust 处理
  nativeOnRenderProcessGone(didCrash: Boolean)
  nativeSslDecision(host: String): Int     0=允许 1=拒绝 2=需询问

AppWebChromeClient
  nativeOnProgressChanged(progress: Int)  用于重置加载超时计时器

AppDownloadListener
  nativeOnDownloadStart(url: String, userAgent: String, contentDisposition: String,
                        mimeType: String, contentLength: Long)
```

`nativeOnInputDialogVisibility` 是必需的：7.2 状态机的第一条分支要求"输入对话框可见时返回优先关闭对话框"，而对话框由 Kotlin 创建，可见性必须回报给持有状态机的 Rust。

`nativeOnRenderProcessGone` 无返回值——Kotlin 侧固定 `return true`（必须接管，否则 Activity 会被系统杀掉），是否重建由 Rust 决定。

### 6.2 Rust 调用的 Kotlin/Android 方法

```
MainActivity
  postToUi()                              任意线程安全，用于跳回 UI 线程
  showInputDialog(prefill: String)        静态/伴生方法，接收 Activity
  showSslPrompt(host: String)             静态/伴生方法，接收 Activity

android.webkit.WebView
  <init>(Context)
  setWebViewClient(WebViewClient) / setWebChromeClient(WebChromeClient)
  setDownloadListener(DownloadListener)
  getSettings() → setJavaScriptEnabled / setDomStorageEnabled / setDatabaseEnabled
                  setUserAgentString / setSupportZoom / setMediaPlaybackRequiresUserGesture
  loadUrl(String) / loadDataWithBaseURL(…) / canGoBack() / goBack() / reload() / destroy()
  setBackgroundColor(int) / setWebContentsDebuggingEnabled(boolean)

android.app.Activity
  setContentView(View) / getFilesDir() / startActivity(Intent) / finish()

android.view.Window / WindowInsetsController
  setDecorFitsSystemWindows(boolean) / hide(statusBars()) / show(statusBars())
  setSystemBarsBehavior(BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE)

android.app.DownloadManager
  Request(Uri) / setDestinationInExternalPublicDir / setNotificationVisibility / enqueue

android.content.Intent
  Intent(ACTION_VIEW, Uri) → startActivity
```

### 6.3 SSL 决策的归属

`SslErrorHandler` 是异步的（等待用户点击对话框），因此**句柄留在 Kotlin**，**策略留在 Rust**：

```
onReceivedSslError 触发
  → Kotlin 调 nativeSslDecision(host)
      → Rust 查允许列表
         命中      → 返回 0 → Kotlin: handler.proceed()
         明确拒绝   → 返回 1 → Kotlin: handler.cancel() + 错误页
         未命中     → 返回 2 → Kotlin: 弹 SslPrompt
                        用户"允许一次"   → handler.proceed()
                        用户"始终允许"   → nativeOnSslChoice(true, true) → handler.proceed()
                        用户"取消"       → nativeOnSslChoice(false, false) → handler.cancel() + 错误页
```

## 7. 交互设计

### 7.1 启动流程

```
冷启动 → Rust 读配置
  ├─ 有地址 → loadUrl，全屏 WebView（不出现任何输入界面）
  └─ 无地址 → WebView 空白在底下，弹输入对话框
```

输入对话框：`EditText` 预填上次地址，软键盘自动弹出；确认 = Rust 校验并规范化 → 写配置 → `loadUrl`。地址非法时不关闭对话框，在输入框下方显示错误。对话框每次显示/关闭都调 `nativeOnInputDialogVisibility` 回报 Rust，因为状态机与页面状态都由 Rust 持有。

### 7.2 返回键状态机

```
短按返回
  ├─ 输入对话框可见        → 关闭对话框，回到网页
  ├─ webView.canGoBack()   → goBack()
  ├─ 已有页面、无历史       → 弹输入对话框
  └─ 已在输入对话框、无历史  → finish()

长按返回 (≥500 ms)
  └─ 无视历史与当前状态      → 直接弹输入对话框
```

长按判定的职责切分，写清楚避免歧义：

- **计时机制在 Kotlin**（阈值常量 `BACK_LONG_PRESS_MS = 500` 也定义在 Kotlin）。因为长按要靠 AndroidX `OnBackPressedCallback` 的回调时序，并用 `BackEventCompat.touchX()` 区分手势与按键（按键时该值按文档返回 NaN）。这是纯粹的 Android 回调管道，放进 Rust 只会更绕。
- **语义在 Rust**。Kotlin 只上报"这是一次短按"或"这是一次长按"两个事件，由 `nav.rs` 决定各自意味着什么。所以 `nav.rs` 的单测覆盖的是"收到长按事件后的状态转移"，**不覆盖 500 ms 计时本身**（见 11 节）。

补充事实：code-server 是单页应用，浏览器 History 几乎不增长，`canGoBack()` 在真实使用中通常立即为 `false`。因此**短按返回按一次即落到输入框**是主路径，长按返回是双保险。

### 7.3 显示与输入

| 项 | 实现 |
|---|---|
| 状态栏 | `WindowInsetsController.hide(statusBars())`，保留导航手势条 |
| 临时唤出系统栏 | `BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE` |
| 软键盘 | `android:windowSoftInputMode="adjustResize"` + edge-to-edge |
| 旋转 | `android:configChanges="orientation|screenSize|keyboardHidden|screenLayout|smallestScreenSize|density|uiMode"`，避免 Activity 重建导致 code-server 重连 |

## 8. 运行时行为

### 8.1 错误处理

| 情况 | 处理 |
|---|---|
| `onReceivedError`（主框架） | Rust 决策，用 `loadDataWithBaseURL` 加载内联错误页（错误码 / URL / 重试按钮），不依赖额外 asset 文件 |
| `onReceivedHttpError` | 同上；对 code-server 的 401 / 502 尤其有用 |
| `onReceivedSslError` | 按 6.3 流程 |
| `onRenderProcessGone` | 必须接管，否则 Activity 被杀。Rust 重建 WebView 并重新加载 |
| 加载超时 | Rust 起一个线程：`loadUrl` 后 20 s 内未收到 `nativeOnPageStarted` 则显示错误页。用 `AtomicBool` 协作；每次 `nativeOnProgressChanged` 重置该计时器，避免大页面被误判 |
| 网络断开 | 交给 code-server 自身重连；应用只保证不销毁 WebView（见 7.3 的 `configChanges`） |

错误页需要一个"重试"入口，但它是 WebView 里的 HTML，无法直接调 Rust。**不用 JS 桥接**，改用自定义 scheme：重试按钮执行 `location.href = 'codeclient://retry'`，该导航进入 `nativeUrlPolicy`，Rust 识别后返回 `2`（已处理）并在内部重新 `loadUrl` 原地址。这样错误页不需要任何桥接类。

### 8.2 文件上传

`onShowFileChooser` 按 `FileChooserParams.getMode()` 分派：

- `MODE_OPEN` → `ActivityResultContracts.GetContent`
- `MODE_OPEN_MULTIPLE` → `OpenMultipleDocuments`
- `MODE_SAVE` → `CreateDocument`

**必须**在用户取消时调用 `filePathCallback.onReceiveValue(null)`，否则下一次点击上传会静默失效。

### 8.3 文件下载

`onDownloadStart` → Rust 经 JNI 调 `DownloadManager`，`setDestinationInExternalPublicDir(DIRECTORY_DOWNLOADS, …)`，带通知。minSdk 29 下无需任何存储权限。

### 8.4 外链

只看 `WebResourceRequest.hasGesture()`：

| `nativeUrlPolicy` 返回 | Kotlin 行为 |
|---|---|
| `0` 内部加载 | `shouldOverrideUrlLoading` 返回 `false`，WebView 自己加载 |
| `1` 跳系统浏览器 | 构造 `ACTION_VIEW` Intent 并 `startActivity`，返回 `true` |
| `2` 已由 Rust 处理 | 返回 `true`，不加载（用于 `codeclient://retry`） |

判定规则：

- `hasGesture() == true` 且跨源 → 返回 `1`（跳系统浏览器）
- `hasGesture() == false`（重定向、JS 跳转）→ 返回 `0`（WebView 内加载）
- scheme 为 `codeclient://` → 返回 `2`（Rust 内部处理）

理由是 code-server 的登录跳转属于重定向，若按"域名不同就跳出"处理会直接打断登录流程。同源判定与白名单在 Rust，Kotlin 只执行 Intent。

### 8.5 地址规范化

| 输入 | 结果 |
|---|---|
| `https://code.example.com` | 原样 |
| `code.example.com` | `https://code.example.com` |
| `192.168.1.10:8080` | `http://192.168.1.10:8080` |
| `http://192.168.1.10:8080` | 原样 |
| `ftp://x` | 拒绝（仅接受 http / https） |
| 空串 | 拒绝 |

主机判定：IPv4 字面量、IPv6 字面量（`[...]` 包裹）视为 IP；其余视为域名。

## 9. 工程结构

```
app-demo/
├─ Cargo.toml                    workspace
├─ rust/                         cdylib crate "codeclient"
│   ├─ Cargo.toml
│   └─ src/{lib,jni_env,app,webview,nav,url,ssl,config,ui,downloads,intents}.rs
├─ android/
│   ├─ settings.gradle.kts / build.gradle.kts / gradle.properties
│   ├─ gradlew / gradle/wrapper/
│   └─ app/
│       ├─ build.gradle.kts / proguard-rules.pro
│       └─ src/main/
│           ├─ AndroidManifest.xml
│           ├─ res/xml/network_security_config.xml
│           ├─ res/values/{strings,themes}.xml
│           ├─ kotlin/io/github/leoyyong/codeclient/
│           │   {MainActivity,AppWebViewClient,AppWebChromeClient,
│           │    AppDownloadListener,InputDialog,SslPrompt,FileChooser}.kt
│           └─ jniLibs/           CI 产物，gitignore
├─ .github/workflows/build.yml
├─ docs/superpowers/specs/2026-09-23-android-vscode-web-client-design.md
└─ README.md
```

Rust 依赖（保持精简）：

- `jni` — JNI 绑定
- `url` — 地址解析与规范化（纯 Rust，可单测）
- `log` + `android_logger` — 输出到 logcat，真机排查必需

线程原语用标准库 `OnceLock` / `Mutex` / `AtomicBool`，不引入额外依赖。

## 10. 构建与 CI

### 10.1 `.github/workflows/build.yml`

```
test job（宿主机，秒级）
  cargo fmt --check
  cargo clippy --all-targets -- -D warnings
  cargo test

apk job
  JDK 17 (temurin)
  Android SDK: platforms;android-35, build-tools;35.0.0, ndk;27.0.12077973
  Rust stable + aarch64-linux-android, armv7-linux-androideabi
  cargo install cargo-ndk
  cargo ndk -t arm64-v8a -t armeabi-v7a -o android/app/src/main/jniLibs build --release
  ./gradlew assembleDebug   (working-directory: android)
  upload-artifact: app-debug.apk
```

用 `assembleDebug` 是因为它由默认 debug keystore 签名，产出可直接侧载安装的 APK，无需配置签名密钥。release 签名留作后续增强。

工具链版本以 CI 为唯一事实来源：上表是基线，若 CI 报告版本不兼容，以 CI 通过的组合为准并回写本文档。

### 10.2 CI 能验证什么、不能验证什么

能验证：

- Rust 编译通过、单测全绿、clippy 无警告
- `.so` 交叉编译成功
- Kotlin 编译通过
- APK 成功产出并可下载

**不能验证**：

- 真机上 WebView 能否连上自签名 code-server
- 长按返回的手感
- 文件选择器能否弹出
- 软键盘是否真的不遮挡编辑区
- 剪贴板读写是否可用

最后这一组只有真机安装一次才能确认。CI 全绿**不构成**"应用可用"的结论。

## 11. 测试策略

TDD 覆盖 Rust 纯逻辑层，四个模块都是无 Android 依赖的纯函数：

| 模块 | 用例要点 |
|---|---|
| `url` | 8.5 表格的每一行；IPv6 字面量；大小写；含路径与查询串；拒绝非 http/https |
| `nav` | 7.2 状态机的每条转移；对话框可见时返回优先关闭对话框；无历史时 `finish()`；收到长按事件时无视历史直接弹输入对话框 |
| `ssl` | 主机大小写不敏感匹配；允许列表命中/未命中；主机名与端口的匹配规则 |
| `config` | 读写往返；文件缺失；文件损坏时回退到默认值而非 panic |

`nav` 接收的是**已判定的事件**（短按 / 长按），不含 500 ms 计时逻辑——那在 Kotlin 侧（见 7.2）。因此 500 ms 阈值本身没有自动化测试覆盖，属于真机手测项。

不写自动化测试的部分：JNI 胶水、Kotlin 层、以及一切需要真实 WebView 的行为。这些靠 CI 编译检查 + 真机手测，并在 README 里列出人工验收清单。

## 12. 非目标（YAGNI）

- 多地址管理 / 历史列表（用户未选）
- 标签页、多窗口
- 密码管理、自动填充
- 主题或深色模式开关（跟随系统）
- iOS、桌面端
- 应用内自动更新
- 上架 Google Play（定位为个人侧载）
- 用前台服务保活 WebSocket（依赖 WebView 存活即可）

## 13. 待真机验证的风险

| 风险 | 若失败的退路 |
|---|---|
| 剪贴板：`navigator.clipboard` 在 Android WebView 支持不完整，读取（粘贴）尤其受限 | 用 `WebChromeClient.onPermissionRequest` 处理 `RESOURCE_CLIPBOARD_READ`；仍不行则加一个 `@JavascriptInterface` 桥接类 |
| 输入框弹出时 code-server 页面被系统回收 | 降低 `android:largeHeap` 之外的内存占用；接受极端情况下的重连 |
| 自签名证书对话框在部分 ROM 上被系统手势吃掉 | 放宽为"默认放行 + 错误页警告" |
| 长按返回在某些 ROM 上与系统手势冲突 | 增加备用手势入口（如双击音量下键） |

## 14. 后续增强（不在本次范围）

- release 构建与签名
- 按主机记忆的证书例外列表管理界面
- 多地址历史与快捷切换
- 自定义 User-Agent（应对 code-server 的浏览器兼容检查）
