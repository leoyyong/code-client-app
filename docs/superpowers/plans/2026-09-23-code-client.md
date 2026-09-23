# Code Client 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use subagent-driven-development (recommended) or executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个 Android 应用，启动时提供地址输入框，确认后以无标题栏、无地址栏的全屏 WebView 显示远端 VS Code Web 端。

**Architecture:** Rust `cdylib` 承载全部决策逻辑（地址校验、配置持久化、返回键状态机、证书策略、下载、外链），通过 JNI 驱动系统 `android.webkit.WebView`。一层约 180 行的 Kotlin 只承担 Android 平台强制要求的部分——继承 `WebViewClient` / `WebChromeClient`、实现 `DownloadListener`、以及启动原生对话框和文件选择管道。

**Tech Stack:** Rust（`jni` / `url` / `log` / `android_logger`）· Kotlin + AndroidX（`AppCompatActivity` / `OnBackPressedCallback` / `ActivityResultContracts`）· Gradle + `cargo-ndk` · GitHub Actions

**Spec:** `docs/superpowers/specs/2026-09-23-android-vscode-web-client-design.md`

## Global Constraints

- **应用 ID**：`io.github.leoyyong.codeclient`（Kotlin 包名同此）
- **Rust crate 名**：`codeclient`，产物为 `libcodeclient.so`
- **minSdk 29，targetSdk 35，compileSdk 35**
- **ABI**：`arm64-v8a`、`armeabi-v7a`
- **Rust 依赖仅限**：`jni`、`url`、`log`、`android_logger`（其中 `android_logger` 只在 `cfg(target_os = "android")` 下引入）。不引入 `serde`、`tokio`、`anyhow` 等
- **线程原语只用标准库**：`OnceLock`、`Mutex`、`AtomicBool`
- **配置文件名**：`codeclient.conf`，位于 `Activity.getFilesDir()`，格式为 `key=value` 文本，`ssl_allow` 可重复
- **长按阈值常量**：`BACK_LONG_PRESS_MS = 500`，定义在 Kotlin 侧
- **证书策略只有两种结果**：`Allow`（允许列表命中）与 `Ask`（需要询问）。不维护拒绝列表
- **错误页重试**用自定义 scheme `codeclient://retry`，不引入 JS 桥接类
- **提交粒度**：每个 Task 结束提交一次，提交信息用 `feat:` / `test:` / `chore:` 前缀
- **本机环境事实**（已实测，2026-09-23）：`cargo` / `rustup` / `rustc` / `gradle` / `adb` / `gh` 全部未安装；Java 只有 8（构建 JDK 需要 17）。**本机没有 Rust 与 Android 工具链**。所有「运行命令并检查输出」的步骤都依赖 Task 0 装好的本机 Rust；APK 的编译与验证依赖 CI
- **`gh` CLI 未安装**：Task 5 起所有 `gh run ...` 命令都需要先 `winget install --id GitHub.cli -e` 并 `gh auth login`，否则改用浏览器打开 Actions 页面查看。计划里的 `gh` 命令不是可选项——没有它就无法确认 CI 结果，而 CI 是唯一的构建验证手段
- **验证边界**：CI 全绿只能证明「编译通过 + 纯逻辑单测通过 + APK 产出」。真机行为（自签名证书、长按手感、文件选择器、软键盘、剪贴板）必须真机手测，见 Task 15 的验收清单

## File Structure

| 文件 | 职责 |
|---|---|
| `Cargo.toml` | workspace 根，仅声明成员 |
| `rust/Cargo.toml` | `codeclient` crate，`crate-type = ["cdylib", "rlib"]` |
| `rust/src/lib.rs` | 模块声明、JNI 导出入口、Android 日志初始化 |
| `rust/src/url.rs` | 地址校验与规范化（纯逻辑） |
| `rust/src/config.rs` | `key=value` 配置读写（纯逻辑 + `std::fs`） |
| `rust/src/ssl.rs` | 证书允许列表判定（纯逻辑） |
| `rust/src/nav.rs` | 返回键状态机（纯逻辑） |
| `rust/src/jni_env.rs` | JNI 封装：`GlobalRef`、`JString` 转换、异常检查 |
| `rust/src/app.rs` | 应用状态：持有 Activity / WebView 全局引用、当前 URL、导航状态 |
| `rust/src/webview.rs` | WebView 创建与操作 |
| `rust/src/ui.rs` | 沉浸式系统栏、屏幕方向 |
| `rust/src/downloads.rs` | `DownloadManager` 调用 |
| `rust/src/intents.rs` | 外链 `ACTION_VIEW` |
| `android/settings.gradle.kts` 等 | Gradle 工程配置 |
| `android/app/src/main/AndroidManifest.xml` | 权限、`configChanges`、`windowSoftInputMode` |
| `android/app/src/main/res/xml/network_security_config.xml` | 明文 + 用户 CA 信任 |
| `android/app/src/main/kotlin/io/github/leoyyong/codeclient/MainActivity.kt` | 生命周期与返回键转发、UI 线程跳板 |
| `.../AppWebViewClient.kt` | 继承 `WebViewClient`，转发 SSL / 导航 / 页面 / 错误 / 渲染崩溃 |
| `.../AppWebChromeClient.kt` | 继承 `WebChromeClient`，转发文件选择 / 进度 |
| `.../AppDownloadListener.kt` | 实现 `DownloadListener`，转发下载 |
| `.../InputDialog.kt` | 地址输入对话框 |
| `.../SslPrompt.kt` | 证书确认对话框 |
| `.../FileChooser.kt` | `ActivityResultContracts` 文件选择管道 |
| `.github/workflows/build.yml` | `test` job + `apk` job |
| `README.md` | 安装步骤、已知限制、人工验收清单 |

---

## Task 0: 工具链与 Cargo workspace 骨架

本机没有 Rust，必须先装。Rust 纯逻辑模块的 TDD 循环（Task 1–4）依赖本机 `cargo test`，否则每个红/绿周期都要等一次 CI。

**Files:**
- Create: `Cargo.toml`
- Create: `rust/Cargo.toml`
- Create: `rust/src/lib.rs`
- Create: `.gitignore`

**Interfaces:**
- Consumes: 无
- Produces: 可运行的 `cargo test`；crate 名 `codeclient`；后续所有 Task 都在这个 workspace 内加模块

- [ ] **Step 1: 安装 Rust 工具链**

Windows 上 MSVC 是默认且受支持最好的目标。约 2–6 GB 下载。

```powershell
winget install --id Rustlang.Rustup -e --accept-source-agreements --accept-package-agreements
winget install --id Microsoft.VisualStudio.2022.BuildTools -e --accept-source-agreements --accept-package-agreements --override "--quiet --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

装完**开一个新 shell**（`PATH` 需要重载），然后验证：

```powershell
cargo --version
rustc --version
```

Expected: 两行版本号，`cargo 1.8x` 或更高。若报 `link.exe not found`，说明 MSVC Build Tools 没装好，先解决再继续。

- [ ] **Step 2: 创建 workspace 根**

`Cargo.toml`：

```toml
[workspace]
members = ["rust"]
resolver = "2"
```

- [ ] **Step 3: 创建 crate 清单**

`rust/Cargo.toml`：

```toml
[package]
name = "codeclient"
version = "0.1.0"
edition = "2021"
publish = false

[lib]
name = "codeclient"
# cdylib 供 Android 通过 JNI 加载；rlib 让宿主机可以链接出测试二进制。
crate-type = ["cdylib", "rlib"]

[dependencies]
jni = "0.21"
url = "2"
log = "0.4"

[target.'cfg(target_os = "android")'.dependencies]
android_logger = "0.14"
```

- [ ] **Step 4: 创建 crate 根模块**

`rust/src/lib.rs`：

```rust
//! Code Client —— 远端 VS Code Web 的 Android 客户端（Rust 侧）。
//!
//! 本 crate 编译为 cdylib 供 Android 通过 JNI 加载，同时编译为 rlib
//! 以便在宿主机上对纯逻辑模块（url / config / ssl / nav）运行单元测试。

#[cfg(test)]
mod sanity {
    #[test]
    fn workspace_builds_and_tests_run() {
        assert_eq!(env!("CARGO_PKG_NAME"), "codeclient");
    }
}
```

- [ ] **Step 5: 创建 .gitignore**

`.gitignore`：

```
/target
/android/.gradle/
/android/build/
/android/app/build/
/android/local.properties
/android/app/src/main/jniLibs/
/.idea/
*.iml
```

`jniLibs/` 必须是忽略项——它是 `cargo-ndk` 的产物，由 CI 生成，绝不能提交。

- [ ] **Step 6: 运行测试，确认骨架可用**

```powershell
cargo test
```

Expected: 编译成功，`test sanity::workspace_builds_and_tests_run ... ok`，`test result: ok. 1 passed`。

- [ ] **Step 7: 提交**

```bash
git add Cargo.toml rust/Cargo.toml rust/src/lib.rs .gitignore
git commit -m "chore: 建立 Cargo workspace 骨架"
```

---

## Task 1: `url.rs` — 地址校验与规范化

**Files:**
- Create: `rust/src/url.rs`
- Modify: `rust/src/lib.rs`（加 `pub mod url;`）
- Test: `rust/src/url.rs` 内的 `#[cfg(test)] mod tests`

**Interfaces:**
- Consumes: 无
- Produces:
  - `pub enum UrlError { Empty, UnsupportedScheme(String), Malformed(String) }`
  - `pub fn normalize(input: &str) -> Result<String, UrlError>`
  - `pub fn host_of(url: &str) -> Option<String>`（返回小写主机名，供 `ssl` 与同源判断使用）

- [ ] **Step 1: 写失败的测试**

在 `rust/src/url.rs` 中写入（此时模块内还没有任何实现，测试必然编译失败）：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn keeps_absolute_https_url() {
        assert_eq!(normalize("https://code.example.com").unwrap(), "https://code.example.com/");
    }

    #[test]
    fn preserves_path_and_query() {
        assert_eq!(normalize("https://x.dev/f?q=1").unwrap(), "https://x.dev/f?q=1");
    }

    #[test]
    fn adds_https_for_bare_domain() {
        assert_eq!(normalize("code.example.com").unwrap(), "https://code.example.com/");
    }

    #[test]
    fn adds_https_for_domain_with_port() {
        assert_eq!(normalize("code.example.com:8080").unwrap(), "https://code.example.com:8080/");
    }

    #[test]
    fn adds_http_for_ipv4_literal() {
        assert_eq!(normalize("192.168.1.10:8080").unwrap(), "http://192.168.1.10:8080/");
    }

    #[test]
    fn adds_http_for_localhost() {
        assert_eq!(normalize("localhost:8080").unwrap(), "http://localhost:8080/");
    }

    #[test]
    fn adds_http_for_ipv6_literal() {
        assert_eq!(normalize("[::1]:8080").unwrap(), "http://[::1]:8080/");
    }

    #[test]
    fn keeps_explicit_http_on_ip() {
        assert_eq!(normalize("http://192.168.1.10:8080").unwrap(), "http://192.168.1.10:8080/");
    }

    #[test]
    fn lowercases_scheme_and_host() {
        assert_eq!(normalize("HTTPS://Code.Example.COM").unwrap(), "https://code.example.com/");
    }

    #[test]
    fn trims_surrounding_whitespace() {
        assert_eq!(normalize("  code.example.com  ").unwrap(), "https://code.example.com/");
    }

    #[test]
    fn rejects_empty_input() {
        assert_eq!(normalize(""), Err(UrlError::Empty));
        assert_eq!(normalize("   "), Err(UrlError::Empty));
    }

    #[test]
    fn rejects_unsupported_scheme() {
        assert_eq!(
            normalize("ftp://x.dev"),
            Err(UrlError::UnsupportedScheme("ftp".to_string()))
        );
    }

    #[test]
    fn rejects_missing_host() {
        assert!(matches!(normalize("https://"), Err(UrlError::Malformed(_))));
    }

    #[test]
    fn host_of_lowercases() {
        assert_eq!(host_of("https://Code.Example.COM/path").as_deref(), Some("code.example.com"));
    }

    #[test]
    fn host_of_returns_none_for_garbage() {
        assert_eq!(host_of("not a url"), None);
    }
}
```

- [ ] **Step 2: 运行测试，确认编译失败**

```powershell
cargo test --package codeclient url
```

Expected: 编译错误，形如 `cannot find function 'normalize' in this scope`。这正是我们要的红灯。

- [ ] **Step 3: 写最小实现**

在 `rust/src/url.rs` 的测试模块**之前**写入：

```rust
use url::Url;

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum UrlError {
    Empty,
    UnsupportedScheme(String),
    Malformed(String),
}

/// 判断输入是否已经是带 scheme 的绝对 URL。
///
/// 必须要求 `://` 分隔符，否则 `192.168.1.10:8080` 里的 `192.168.1.10`
/// 会被误当成 scheme —— 而 RFC 里 scheme 不允许以数字开头，这正好用来区分。
fn has_scheme(input: &str) -> bool {
    match input.find("://") {
        Some(i) => {
            let s = &input[..i];
            let mut chars = s.chars();
            match chars.next() {
                Some(c) if c.is_ascii_alphabetic() => {
                    chars.all(|c| c.is_ascii_alphanumeric() || c == '+' || c == '-' || c == '.')
                }
                _ => false,
            }
        }
        None => false,
    }
}

/// 取出 host[:port] 部分，用于在没有 scheme 时决定默认协议。
fn host_port_part(input: &str) -> &str {
    let rest = match input.find("://") {
        Some(i) => &input[i + 3..],
        None => input,
    };
    match rest.find(['/', '?', '#']) {
        Some(i) => &rest[..i],
        None => rest,
    }
}

/// IP 字面量、IPv6 字面量和 `localhost` 默认走明文 http；
/// 其余视为域名，默认走 https。
fn defaults_to_plain_http(host_port: &str) -> bool {
    if host_port.starts_with('[') {
        return true;
    }
    let host = host_port.split(':').next().unwrap_or("");
    if host.eq_ignore_ascii_case("localhost") {
        return true;
    }
    let octets: Vec<&str> = host.split('.').collect();
    octets.len() == 4
        && octets
            .iter()
            .all(|o| !o.is_empty() && o.len() <= 3 && o.chars().all(|c| c.is_ascii_digit()))
}

/// 校验并规范化用户输入的地址。
pub fn normalize(input: &str) -> Result<String, UrlError> {
    let trimmed = input.trim();
    if trimmed.is_empty() {
        return Err(UrlError::Empty);
    }

    let candidate = if has_scheme(trimmed) {
        trimmed.to_string()
    } else if defaults_to_plain_http(host_port_part(trimmed)) {
        format!("http://{trimmed}")
    } else {
        format!("https://{trimmed}")
    };

    let parsed = Url::parse(&candidate).map_err(|e| UrlError::Malformed(e.to_string()))?;

    match parsed.scheme() {
        "http" | "https" => {}
        other => return Err(UrlError::UnsupportedScheme(other.to_string())),
    }

    if parsed.host_str().is_none() {
        return Err(UrlError::Malformed("缺少主机名".to_string()));
    }

    Ok(parsed.to_string())
}

/// 取规范化后的小写主机名。用于证书允许列表匹配与外链同源判断。
pub fn host_of(url: &str) -> Option<String> {
    Url::parse(url)
        .ok()
        .and_then(|u| u.host_str().map(|h| h.to_ascii_lowercase()))
}
```

同时在 `rust/src/lib.rs` 顶部加入模块声明：

```rust
pub mod url;
```

- [ ] **Step 4: 运行测试，确认全部通过**

```powershell
cargo test --package codeclient url
```

Expected: `test result: ok. 15 passed`。

- [ ] **Step 5: 提交**

```bash
git add rust/src/url.rs rust/src/lib.rs
git commit -m "feat: 地址校验与规范化"
```

---

## Task 2: `config.rs` — 配置读写

格式选 `key=value` 纯文本而非 JSON，是为了不引入 `serde`（Global Constraints 限定了依赖集合）。URL 里可能含 `=`，所以解析用 `split_once` 只切第一个。

**Files:**
- Create: `rust/src/config.rs`
- Modify: `rust/src/lib.rs`（加 `pub mod config;`）
- Test: `rust/src/config.rs` 内的 `#[cfg(test)] mod tests`

**Interfaces:**
- Consumes: 无
- Produces:
  - `pub const FILE_NAME: &str = "codeclient.conf"`
  - `pub struct Config { pub url: Option<String>, pub ssl_allow: Vec<String> }`（`Default` + `PartialEq`）
  - `pub fn load(dir: &Path) -> Config`
  - `pub fn save(dir: &Path, cfg: &Config) -> std::io::Result<()>`

- [ ] **Step 1: 写失败的测试**

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::fs;

    fn tmpdir(tag: &str) -> std::path::PathBuf {
        let d = std::env::temp_dir().join(format!("codeclient-test-{tag}"));
        let _ = fs::remove_dir_all(&d);
        fs::create_dir_all(&d).unwrap();
        d
    }

    #[test]
    fn round_trips_url_and_allowlist() {
        let dir = tmpdir("roundtrip");
        let cfg = Config {
            url: Some("https://code.example.com/".to_string()),
            ssl_allow: vec!["code.example.com".to_string(), "192.168.1.10".to_string()],
        };
        save(&dir, &cfg).unwrap();
        assert_eq!(load(&dir), cfg);
    }

    #[test]
    fn missing_file_yields_default() {
        let dir = tmpdir("missing");
        assert_eq!(load(&dir), Config::default());
    }

    #[test]
    fn corrupt_file_yields_default_without_panic() {
        let dir = tmpdir("corrupt");
        fs::write(dir.join(FILE_NAME), "!!!\n\u{0}\nno equals sign here\n").unwrap();
        assert_eq!(load(&dir), Config::default());
    }

    #[test]
    fn partial_corruption_keeps_valid_lines() {
        let dir = tmpdir("partial");
        fs::write(dir.join(FILE_NAME), "garbage\nurl=https://a.dev/\n???\n").unwrap();
        let cfg = load(&dir);
        assert_eq!(cfg.url.as_deref(), Some("https://a.dev/"));
        assert!(cfg.ssl_allow.is_empty());
    }

    #[test]
    fn url_containing_equals_sign_round_trips() {
        let dir = tmpdir("equals");
        let cfg = Config {
            url: Some("https://x.dev/f?a=b=c".to_string()),
            ssl_allow: vec![],
        };
        save(&dir, &cfg).unwrap();
        assert_eq!(load(&dir).url.as_deref(), Some("https://x.dev/f?a=b=c"));
    }

    #[test]
    fn allowlist_order_is_preserved() {
        let dir = tmpdir("order");
        let cfg = Config {
            url: None,
            ssl_allow: vec!["b.dev".to_string(), "a.dev".to_string()],
        };
        save(&dir, &cfg).unwrap();
        assert_eq!(load(&dir).ssl_allow, vec!["b.dev".to_string(), "a.dev".to_string()]);
    }

    #[test]
    fn allowlist_entries_are_trimmed_and_lowercased() {
        let dir = tmpdir("trim");
        fs::write(dir.join(FILE_NAME), "ssl_allow=  Code.Example.COM  \n").unwrap();
        assert_eq!(load(&dir).ssl_allow, vec!["code.example.com".to_string()]);
    }

    #[test]
    fn empty_allowlist_entry_is_dropped() {
        let dir = tmpdir("emptyentry");
        fs::write(dir.join(FILE_NAME), "ssl_allow=\nssl_allow=a.dev\n").unwrap();
        assert_eq!(load(&dir).ssl_allow, vec!["a.dev".to_string()]);
    }

    #[test]
    fn save_does_not_leave_temp_file() {
        let dir = tmpdir("notmp");
        save(&dir, &Config::default()).unwrap();
        assert!(!dir.join("codeclient.conf.tmp").exists());
    }
}
```

- [ ] **Step 2: 运行测试，确认编译失败**

```powershell
cargo test --package codeclient config
```

Expected: `cannot find function 'load' in this scope`。

- [ ] **Step 3: 写最小实现**

```rust
use std::fs;
use std::io;
use std::path::Path;

pub const FILE_NAME: &str = "codeclient.conf";
const TMP_NAME: &str = "codeclient.conf.tmp";

#[derive(Debug, Clone, PartialEq, Eq, Default)]
pub struct Config {
    pub url: Option<String>,
    pub ssl_allow: Vec<String>,
}

/// 读取配置。文件缺失或内容无法解析时返回默认值——配置损坏不应该让应用崩溃。
pub fn load(dir: &Path) -> Config {
    let Ok(text) = fs::read_to_string(dir.join(FILE_NAME)) else {
        return Config::default();
    };

    let mut cfg = Config::default();
    for line in text.lines() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let Some((key, value)) = line.split_once('=') else {
            continue;
        };
        match key.trim() {
            "url" => {
                let v = value.trim();
                if !v.is_empty() {
                    cfg.url = Some(v.to_string());
                }
            }
            "ssl_allow" => {
                let v = value.trim().to_ascii_lowercase();
                if !v.is_empty() {
                    cfg.ssl_allow.push(v);
                }
            }
            _ => {}
        }
    }
    cfg
}

/// 写配置。先写临时文件再 rename，避免写到一半断电留下半个文件。
pub fn save(dir: &Path, cfg: &Config) -> io::Result<()> {
    let mut out = String::new();
    if let Some(url) = &cfg.url {
        out.push_str("url=");
        out.push_str(url);
        out.push('\n');
    }
    for host in &cfg.ssl_allow {
        out.push_str("ssl_allow=");
        out.push_str(host);
        out.push('\n');
    }

    let tmp = dir.join(TMP_NAME);
    fs::write(&tmp, out)?;
    fs::rename(&tmp, dir.join(FILE_NAME))
}
```

同时在 `rust/src/lib.rs` 加入 `pub mod config;`。

- [ ] **Step 4: 运行测试，确认全部通过**

```powershell
cargo test --package codeclient config
```

Expected: `test result: ok. 9 passed`。

- [ ] **Step 5: 提交**

```bash
git add rust/src/config.rs rust/src/lib.rs
git commit -m "feat: 配置读写"
```

---

## Task 3: `ssl.rs` — 证书允许列表判定

**Files:**
- Create: `rust/src/ssl.rs`
- Modify: `rust/src/lib.rs`（加 `pub mod ssl;`）
- Test: `rust/src/ssl.rs` 内的 `#[cfg(test)] mod tests`

**Interfaces:**
- Consumes: 无
- Produces:
  - `pub enum SslDecision { Allow, Ask }`
  - `pub fn decide(allow: &[String], host: &str) -> SslDecision`
  - `pub fn remember(allow: &mut Vec<String>, host: &str)`

- [ ] **Step 1: 写失败的测试**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn empty_allowlist_asks() {
        assert_eq!(decide(&[], "code.example.com"), SslDecision::Ask);
    }

    #[test]
    fn exact_match_allows() {
        let allow = vec!["code.example.com".to_string()];
        assert_eq!(decide(&allow, "code.example.com"), SslDecision::Allow);
    }

    #[test]
    fn match_is_case_insensitive() {
        let allow = vec!["code.example.com".to_string()];
        assert_eq!(decide(&allow, "Code.Example.COM"), SslDecision::Allow);
    }

    #[test]
    fn unmatched_host_asks() {
        let allow = vec!["a.dev".to_string()];
        assert_eq!(decide(&allow, "b.dev"), SslDecision::Ask);
    }

    #[test]
    fn empty_host_asks() {
        let allow = vec!["a.dev".to_string()];
        assert_eq!(decide(&allow, ""), SslDecision::Ask);
    }

    #[test]
    fn port_is_not_part_of_the_match() {
        let allow = vec!["a.dev".to_string()];
        assert_eq!(decide(&allow, "a.dev:8443"), SslDecision::Ask);
    }

    #[test]
    fn remember_is_idempotent() {
        let mut allow = vec![];
        remember(&mut allow, "A.dev");
        remember(&mut allow, "a.DEV");
        assert_eq!(allow, vec!["a.dev".to_string()]);
    }

    #[test]
    fn remember_ignores_empty_host() {
        let mut allow = vec![];
        remember(&mut allow, "  ");
        assert!(allow.is_empty());
    }
}
```

- [ ] **Step 2: 运行测试，确认编译失败**

```powershell
cargo test --package codeclient ssl
```

Expected: `cannot find function 'decide' in this scope`。

- [ ] **Step 3: 写最小实现**

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum SslDecision {
    /// 该主机已在允许列表中，直接 proceed()。
    Allow,
    /// 未命中，需要弹对话框询问用户。
    Ask,
}

/// 判定证书错误该如何处理。
///
/// `host` 取自 SslError 的 URL 主机名。端口不参与匹配：
/// `SslError.getUrl().getHost()` 本身就不含端口。
pub fn decide(allow: &[String], host: &str) -> SslDecision {
    let host = host.trim().to_ascii_lowercase();
    if host.is_empty() {
        return SslDecision::Ask;
    }
    if allow.iter().any(|h| h == &host) {
        SslDecision::Allow
    } else {
        SslDecision::Ask
    }
}

/// 把主机加入允许列表。大小写归一后去重。
pub fn remember(allow: &mut Vec<String>, host: &str) {
    let host = host.trim().to_ascii_lowercase();
    if host.is_empty() || allow.iter().any(|h| h == &host) {
        return;
    }
    allow.push(host);
}
```

同时在 `rust/src/lib.rs` 加入 `pub mod ssl;`。

- [ ] **Step 4: 运行测试，确认全部通过**

```powershell
cargo test --package codeclient ssl
```

Expected: `test result: ok. 8 passed`。

- [ ] **Step 5: 提交**

```bash
git add rust/src/ssl.rs rust/src/lib.rs
git commit -m "feat: 证书允许列表判定"
```

---

## Task 4: `nav.rs` — 返回键状态机

这是整个应用里唯一有真实分支逻辑的地方，也是 Spec 7.2 那张表的可执行版本。

**Files:**
- Create: `rust/src/nav.rs`
- Modify: `rust/src/lib.rs`（加 `pub mod nav;`）
- Test: `rust/src/nav.rs` 内的 `#[cfg(test)] mod tests`

**Interfaces:**
- Consumes: 无
- Produces:
  - `pub enum BackEvent { ShortPress, LongPress }`
  - `pub struct NavState { pub dialog_visible: bool, pub can_go_back: bool, pub has_page: bool }`
  - `pub enum NavAction { DismissDialog, GoBack, ShowInput, Finish }`
  - `pub fn decide(state: NavState, event: BackEvent) -> NavAction`

- [ ] **Step 1: 写失败的测试**

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn state(dialog_visible: bool, can_go_back: bool, has_page: bool) -> NavState {
        NavState { dialog_visible, can_go_back, has_page }
    }

    #[test]
    fn short_press_table_is_total_and_matches_spec() {
        // Spec 7.2：六个分支互斥且穷尽。
        // can_go_back=true 而 has_page=false 在真实运行中不会出现（没有页面就
        // 没有历史），但表里仍然给它一个确定结果，避免出现未定义行为。
        let cases = [
            ((true, true, true), NavAction::DismissDialog),
            ((true, true, false), NavAction::Finish),
            ((true, false, true), NavAction::DismissDialog),
            ((true, false, false), NavAction::Finish),
            ((false, true, true), NavAction::GoBack),
            ((false, true, false), NavAction::GoBack),
            ((false, false, true), NavAction::ShowInput),
            ((false, false, false), NavAction::Finish),
        ];
        for ((dialog, go_back, page), expected) in cases {
            let s = state(dialog, go_back, page);
            assert_eq!(decide(s, BackEvent::ShortPress), expected, "state = {s:?}");
        }
    }

    #[test]
    fn long_press_always_shows_input() {
        for dialog in [false, true] {
            for go_back in [false, true] {
                for page in [false, true] {
                    let s = state(dialog, go_back, page);
                    assert_eq!(decide(s, BackEvent::LongPress), NavAction::ShowInput, "state = {s:?}");
                }
            }
        }
    }

    #[test]
    fn dialog_dismiss_wins_over_history() {
        // 对话框开着时，返回优先关对话框，而不是让网页后退。
        let s = state(true, true, true);
        assert_eq!(decide(s, BackEvent::ShortPress), NavAction::DismissDialog);
    }

    #[test]
    fn first_launch_with_dialog_open_finishes() {
        // 冷启动、没加载过任何页面、输入框开着：返回应该退出应用，
        // 而不是关掉对话框留下一片空白。
        let s = state(true, false, false);
        assert_eq!(decide(s, BackEvent::ShortPress), NavAction::Finish);
    }
}
```

- [ ] **Step 2: 运行测试，确认编译失败**

```powershell
cargo test --package codeclient nav
```

Expected: `cannot find function 'decide' in this scope`。

- [ ] **Step 3: 写最小实现**

```rust
/// Kotlin 侧上报的返回键事件。长按判定（500 ms 阈值）在 Kotlin 完成，
/// 这里只接收已经判定好的结果。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BackEvent {
    ShortPress,
    LongPress,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct NavState {
    /// 地址输入对话框当前是否可见。
    pub dialog_visible: bool,
    /// WebView 是否有可回退的历史。
    pub can_go_back: bool,
    /// 是否已经加载过页面（决定输入框关掉之后有没有内容可看）。
    pub has_page: bool,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum NavAction {
    DismissDialog,
    GoBack,
    ShowInput,
    Finish,
}

/// Spec 7.2 的返回键状态机。
pub fn decide(state: NavState, event: BackEvent) -> NavAction {
    if event == BackEvent::LongPress {
        return NavAction::ShowInput;
    }

    if state.dialog_visible {
        return if state.has_page {
            NavAction::DismissDialog
        } else {
            NavAction::Finish
        };
    }

    if state.can_go_back {
        return NavAction::GoBack;
    }

    if state.has_page {
        NavAction::ShowInput
    } else {
        NavAction::Finish
    }
}
```

同时在 `rust/src/lib.rs` 加入 `pub mod nav;`。

- [ ] **Step 4: 运行测试，确认全部通过**

```powershell
cargo test --package codeclient
```

Expected: `test result: ok. 33 passed`（1 sanity + 15 url + 9 config + 8 ssl + 4 nav，其中 `short_press_table_is_total_and_matches_spec` 内含 8 个断言但算 1 个测试）。

- [ ] **Step 5: 提交**

```bash
git add rust/src/nav.rs rust/src/lib.rs
git commit -m "feat: 返回键状态机"
```

---

## Task 5: Android Gradle 工程骨架与 CI

目标是在接入任何 Rust 代码之前，先让 CI 能产出一个**能装能启动的空壳 APK**。这样后面每个 Task 的失败都能明确归因到新增代码，而不是工程配置。

**Files:**
- Create: `android/settings.gradle.kts`
- Create: `android/build.gradle.kts`
- Create: `android/gradle.properties`
- Create: `android/app/build.gradle.kts`
- Create: `android/app/src/main/AndroidManifest.xml`
- Create: `android/app/src/main/res/xml/network_security_config.xml`
- Create: `android/app/src/main/res/values/strings.xml`
- Create: `android/app/src/main/res/values/themes.xml`
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/MainActivity.kt`
- Create: `.github/workflows/build.yml`

**Interfaces:**
- Consumes: Task 0 的 Cargo workspace（CI 的 `test` job 要跑它）
- Produces: 可安装的 `app-debug.apk`；`MainActivity` 类名 `io.github.leoyyong.codeclient.MainActivity`（JNI 导出符号依赖这个完全限定名）

- [ ] **Step 1: 创建 Gradle 工程配置**

`android/settings.gradle.kts`：

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "CodeClient"
include(":app")
```

`android/build.gradle.kts`：

```kotlin
plugins {
    id("com.android.application") version "8.7.3" apply false
    id("org.jetbrains.kotlin.android") version "2.0.21" apply false
}
```

`android/gradle.properties`：

```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
kotlin.code.style=official
```

**刻意不提交 Gradle wrapper**：`gradle-wrapper.jar` 是二进制，本机没有 Gradle 无法生成它，手写一个不可信的 jar 比不写更糟。CI 用 `gradle/actions/setup-gradle` 固定版本调用 `gradle`。本机需要构建时再跑 `gradle wrapper` 生成。

- [ ] **Step 2: 创建 app 模块配置**

`android/app/build.gradle.kts`：

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "io.github.leoyyong.codeclient"
    compileSdk = 35

    defaultConfig {
        applicationId = "io.github.leoyyong.codeclient"
        minSdk = 29
        targetSdk = 35
        versionCode = 1
        versionName = "0.1.0"
        // 只打这两个 ABI，与 CI 里 cargo-ndk 的 -t 参数保持一致。
        ndk { abiFilters += listOf("arm64-v8a", "armeabi-v7a") }
    }

    buildTypes {
        release { isMinifyEnabled = false }
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions { jvmTarget = "17" }

    sourceSets["main"].java.srcDir("src/main/kotlin")
}

dependencies {
    implementation("androidx.appcompat:appcompat:1.7.0")
    implementation("androidx.activity:activity-ktx:1.9.3")
    implementation("androidx.core:core-ktx:1.15.0")
}
```

**刻意不声明 `android:icon`**：图标需要二进制 PNG 资源，本计划不生成二进制文件。系统会退化到默认图标。加真图标是 Task 15 之后的事。

- [ ] **Step 3: 创建 manifest**

`android/app/src/main/AndroidManifest.xml`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

    <application
        android:label="@string/app_name"
        android:supportsRtl="true"
        android:usesCleartextTraffic="true"
        android:networkSecurityConfig="@xml/network_security_config"
        android:enableOnBackInvokedCallback="true"
        android:theme="@style/Theme.CodeClient">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTask"
            android:configChanges="orientation|screenSize|keyboardHidden|screenLayout|smallestScreenSize|density|uiMode"
            android:windowSoftInputMode="adjustResize">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

三个属性各有明确理由，删任何一个都会破坏 Spec 里的行为：

- `configChanges`：避免旋转时重建 Activity 导致 code-server 重连（Spec 7.3）
- `windowSoftInputMode="adjustResize"`：软键盘不遮挡编辑区（Spec 7.3）
- `enableOnBackInvokedCallback="true"`：predictive back，长按返回依赖它（Spec 7.2）

- [ ] **Step 4: 创建资源**

`android/app/src/main/res/xml/network_security_config.xml`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <!-- 局域网 code-server 常走明文 http -->
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
            <!-- 允许用户自装的 CA，便于私有 PKI -->
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

`android/app/src/main/res/values/strings.xml`：

```xml
<resources>
    <string name="app_name">Code Client</string>
    <string name="input_title">远端 VS Code 地址</string>
    <string name="input_hint">code.example.com 或 192.168.1.10:8080</string>
    <string name="input_confirm">打开</string>
    <string name="input_cancel">取消</string>
    <string name="ssl_title">证书无法验证</string>
    <string name="ssl_message">%1$s 的证书不受信任。继续可能让你的会话被第三方截获。</string>
    <string name="ssl_allow_once">仅本次允许</string>
    <string name="ssl_allow_always">始终允许此主机</string>
    <string name="ssl_cancel">取消</string>
    <string name="error_title">无法加载页面</string>
</resources>
```

`android/app/src/main/res/values/themes.xml`：

```xml
<resources>
    <style name="Theme.CodeClient" parent="Theme.AppCompat.DayNight.NoActionBar" />
</resources>
```

`NoActionBar` 是「不带标题栏」这条需求的落点。

- [ ] **Step 5: 创建最小 MainActivity**

`android/app/src/main/kotlin/io/github/leoyyong/codeclient/MainActivity.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
    }
}
```

- [ ] **Step 6: 创建 CI workflow**

`.github/workflows/build.yml`：

```yaml
name: build

on:
  push:
    branches: [master]
  pull_request:
  workflow_dispatch:

env:
  NDK_VERSION: 27.0.12077973

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test --all

  apk:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - uses: android-actions/setup-android@v3

      - name: 安装 Android SDK 与 NDK
        run: sdkmanager "platforms;android-35" "build-tools;35.0.0" "ndk;$NDK_VERSION"

      - name: 导出 NDK 路径
        run: echo "ANDROID_NDK_HOME=$ANDROID_HOME/ndk/$NDK_VERSION" >> "$GITHUB_ENV"

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: aarch64-linux-android,armv7-linux-androideabi

      - uses: Swatinem/rust-cache@v2

      - name: 安装 cargo-ndk
        run: cargo install cargo-ndk --locked

      - name: 交叉编译 Rust
        run: cargo ndk -t arm64-v8a -t armeabi-v7a -o android/app/src/main/jniLibs build --release

      - uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: 打包 APK
        working-directory: android
        run: gradle assembleDebug --no-daemon

      - uses: actions/upload-artifact@v4
        with:
          name: app-debug
          path: android/app/build/outputs/apk/debug/app-debug.apk
          if-no-files-found: error
```

`if-no-files-found: error` 是刻意的：APK 没产出就必须让 job 失败，而不是上传一个空 artifact 让流水线看起来是绿的。

- [ ] **Step 7: 提交并推送**

```bash
git add android .github
git commit -m "chore: Android Gradle 工程骨架与 CI"
git push origin master
```

- [ ] **Step 8: 等待 CI，确认两个 job 都绿**

```powershell
gh run list --workflow=build --limit 1
gh run watch
```

Expected: `test` 与 `apk` 均 `completed / success`。

若失败，按 job 归因：

| 失败位置 | 最可能原因 |
|---|---|
| `test` job 的 clippy | Task 1–4 的代码有 lint 问题，按提示改 |
| `sdkmanager` | 包名或 NDK 版本号不对，用 `sdkmanager --list` 查可用版本 |
| `cargo ndk` | `ANDROID_NDK_HOME` 没生效，检查上一步是否写进了 `$GITHUB_ENV` |
| `gradle assembleDebug` | AGP / Gradle / Kotlin 版本组合不兼容。这是最可能出问题的一步，调整 `android/build.gradle.kts` 里的版本号直到通过，**并把最终可用的组合回写到 Spec 10.1 节** |

- [ ] **Step 9: 确认 artifact 存在**

```powershell
gh run download --name app-debug --dir /tmp/apk-check
```

Expected: 下载到 `app-debug.apk`，体积约 2–4 MB。此时它还是个空壳，装上只会显示空白界面——这是预期的。

---

## Task 6: Rust ↔ Kotlin JNI 打通

目标：Rust 拿到 Activity 引用并创建一个 WebView 填满窗口，加载一个**硬编码**地址。地址输入留到 Task 7。

**Files:**
- Create: `rust/src/jni_env.rs`
- Create: `rust/src/app.rs`
- Create: `rust/src/webview.rs`
- Modify: `rust/src/lib.rs`（模块声明 + JNI 导出）
- Modify: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/MainActivity.kt`

**Interfaces:**
- Consumes: Task 0 的 crate
- Produces:
  - Rust 导出符号（供后续所有 Task 使用）：
    - `Java_io_github_leoyyong_codeclient_MainActivity_nativeCreate(env, class, activity: JObject, filesDir: JString)`
    - `Java_io_github_leoyyong_codeclient_MainActivity_nativeOnResume/nativeOnPause/nativeOnDestroy`
  - `pub struct AppState { pub activity: GlobalRef, pub webview: Option<GlobalRef>, pub files_dir: String, pub config: Config, pub nav: NavState }`
  - `pub fn slot() -> &'static Mutex<Option<AppState>>`
  - `jni_env::jstring_to_string(env: &mut JNIEnv, s: &JString) -> Option<String>`
  - `webview::create(env: &mut JNIEnv, activity: &JObject) -> jni::errors::Result<GlobalRef>`

- [ ] **Step 1: 写 JNI 工具模块**

`rust/src/jni_env.rs`：

```rust
use jni::objects::{JObject, JString, JValue};
use jni::JNIEnv;
use jni::errors::Result as JniResult;

/// 把 Java String 转成 Rust String。转换失败时返回 None 而不是 panic ——
/// 从 JNI 边界 panic 会直接终止进程。
pub fn jstring_to_string(env: &mut JNIEnv, s: &JString) -> Option<String> {
    if s.is_null() {
        return None;
    }
    env.get_string(s).ok().map(|js| js.to_string_lossy().into_owned())
}

/// 调用无参 void 方法，并吞掉异常。
///
/// 每个 JNI 调用后都必须检查异常：如果不清理，异常会一直挂在线程上，
/// 下一个 JNI 调用会以莫名其妙的方式失败。
pub fn call_void(env: &mut JNIEnv, obj: &JObject, name: &str) {
    if let Err(e) = env.call_method(obj, name, "()V", &[]) {
        log::warn!("调用 {name} 失败: {e}");
        let _ = env.exception_clear();
    }
}

/// 调用返回 boolean 的无参方法；失败时返回 fallback。
pub fn call_bool(env: &mut JNIEnv, obj: &JObject, name: &str) -> bool {
    match env.call_method(obj, name, "()Z", &[]) {
        Ok(v) => v.z().unwrap_or(false),
        Err(e) => {
            log::warn!("调用 {name} 失败: {e}");
            let _ = env.exception_clear();
            false
        }
    }
}

/// 调用返回 int 的无参方法；失败时返回 fallback。
pub fn call_int(env: &mut JNIEnv, obj: &JObject, name: &str) -> i32 {
    match env.call_method(obj, name, "()I", &[]) {
        Ok(v) => v.i().unwrap_or(-1),
        Err(e) => {
            log::warn!("调用 {name} 失败: {e}");
            let _ = env.exception_clear();
            -1
        }
    }
}

/// 调用接收单个 String 的 void 方法。
pub fn call_void_str(env: &mut JNIEnv, obj: &JObject, name: &str, arg: &str) {
    let Ok(jarg) = env.new_string(arg) else {
        log::warn!("构造 Java String 失败");
        return;
    };
    if let Err(e) = env.call_method(
        obj,
        name,
        "(Ljava/lang/String;)V",
        &[JValue::Object(&JObject::from(jarg))],
    ) {
        log::warn!("调用 {name} 失败: {e}");
        let _ = env.exception_clear();
    }
}
```

- [ ] **Step 2: 写应用状态模块**

`rust/src/app.rs`：

```rust
use crate::config::Config;
use crate::nav::NavState;
use jni::objects::GlobalRef;
use std::sync::{Mutex, OnceLock};

pub struct AppState {
    pub activity: GlobalRef,
    pub webview: Option<GlobalRef>,
    pub files_dir: String,
    pub config: Config,
    pub nav: NavState,
    /// 用户当前要连接的目标地址（规范化之后）。
    pub target_url: Option<String>,
}

static STATE: OnceLock<Mutex<Option<AppState>>> = OnceLock::new();

pub fn slot() -> &'static Mutex<Option<AppState>> {
    STATE.get_or_init(|| Mutex::new(None))
}

/// 在持有锁的前提下对状态做一次操作。返回值表示状态是否存在。
pub fn with_state<R>(f: impl FnOnce(&mut AppState) -> R) -> Option<R> {
    let mut guard = slot().lock().ok()?;
    guard.as_mut().map(f)
}
```

**如果 `GlobalRef` 不满足 `Send`**（jni 0.21 中它应当是 `Send + Sync`，但以编译器为准），`static` 会因为 `AppState: !Send` 而拒绝编译。届时按顺序尝试：先确认 `global_ref` feature 已默认启用；若确实不满足，把 `activity` / `webview` 改存 `jobject` 裸指针，并另行在 Rust 侧维护引用计数与释放。**不要**用 `unsafe impl Send` 绕过——那会把未定义行为藏起来。

- [ ] **Step 3: 写 WebView 创建模块**

`rust/src/webview.rs`：

```rust
use jni::errors::Result as JniResult;
use jni::objects::{GlobalRef, JObject, JValue};
use jni::JNIEnv;

/// 创建一个填满 Activity 窗口的 WebView，并返回它的全局引用。
///
/// WebView 只能在 UI 线程创建；`nativeCreate` 由 `Activity.onCreate` 调用，
/// 天然在 UI 线程上。
pub fn create(env: &mut JNIEnv, activity: &JObject) -> JniResult<GlobalRef> {
    let webview = env.new_object(
        "android/webkit/WebView",
        "(Landroid/content/Context;)V",
        &[JValue::Object(activity)],
    )?;

    let settings = env
        .call_method(&webview, "getSettings", "()Landroid/webkit/WebSettings;", &[])?
        .l()?;
    for (setter, sig) in [
        ("setJavaScriptEnabled", "(Z)V"),
        ("setDomStorageEnabled", "(Z)V"),
        ("setDatabaseEnabled", "(Z)V"),
    ] {
        env.call_method(&settings, setter, sig, &[JValue::Bool(1)])?;
    }
    // code-server 会检查 UA；带上 Chrome 标识避免被判定为不受支持的浏览器。
    let ua = env.new_string("Mozilla/5.0 (Linux; Android 14) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36")?;
    env.call_method(
        &settings,
        "setUserAgentString",
        "(Ljava/lang/String;)V",
        &[JValue::Object(&JObject::from(ua))],
    )?;

    // 让 WebView 自身可调焦，否则软键盘有时不会弹出。
    env.call_method(&webview, "setFocusable", "(Z)V", &[JValue::Bool(1)])?;
    env.call_method(&webview, "setFocusableInTouchMode", "(Z)V", &[JValue::Bool(1)])?;

    env.call_method(
        activity,
        "setContentView",
        "(Landroid/view/View;)V",
        &[JValue::Object(&webview)],
    )?;

    env.new_global_ref(&webview)
}

/// 在 WebView 里加载地址。
pub fn load_url(env: &mut JNIEnv, webview: &JObject, url: &str) -> JniResult<()> {
    let jurl = env.new_string(url)?;
    env.call_method(
        webview,
        "loadUrl",
        "(Ljava/lang/String;)V",
        &[JValue::Object(&JObject::from(jurl))],
    )?;
    Ok(())
}

pub fn can_go_back(env: &mut JNIEnv, webview: &JObject) -> bool {
    env.call_method(webview, "canGoBack", "()Z", &[])
        .and_then(|v| v.z())
        .unwrap_or(false)
}

pub fn go_back(env: &mut JNIEnv, webview: &JObject) {
    let _ = env.call_method(webview, "goBack", "()V", &[]);
}
```

- [ ] **Step 4: 写 lib.rs 的 JNI 导出**

`rust/src/lib.rs` 完整替换为：

```rust
//! Code Client —— 远端 VS Code Web 的 Android 客户端（Rust 侧）。

pub mod app;
pub mod config;
pub mod jni_env;
pub mod nav;
pub mod ssl;
pub mod url;
pub mod webview;

use jni::objects::{JClass, JObject, JString};
use jni::JNIEnv;

mod android_log {
    #[cfg(target_os = "android")]
    pub fn init() {
        android_logger::init_once(
            android_logger::Config::default().with_max_level(log::LevelFilter::Debug),
        );
    }

    #[cfg(not(target_os = "android"))]
    pub fn init() {}
}

/// 启动时调用一次。`activity` 由 Kotlin 传入，`filesDir` 用于读写配置。
#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeCreate(
    mut env: JNIEnv,
    _class: JClass,
    activity: JObject,
    files_dir: JString,
) {
    android_log::init();
    log::info!("nativeCreate 被调用");

    let Some(files_dir) = jni_env::jstring_to_string(&mut env, &files_dir) else {
        log::error!("filesDir 为空，放弃初始化");
        return;
    };

    let Ok(activity_ref) = env.new_global_ref(&activity) else {
        log::error!("无法持有 Activity 全局引用");
        return;
    };

    let config = config::load(std::path::Path::new(&files_dir));

    let Ok(webview_ref) = webview::create(&mut env, &activity) else {
        log::error!("创建 WebView 失败");
        let _ = env.exception_clear();
        return;
    };

    // Task 6 临时硬编码；Task 7 改成从 config 读取并由输入对话框设置。
    const PLACEHOLDER_URL: &str = "https://example.com/";
    if let Err(e) = webview::load_url(&mut env, &webview_ref.as_obj(), PLACEHOLDER_URL) {
        log::error!("加载地址失败: {e}");
        let _ = env.exception_clear();
    }

    let state = app::AppState {
        activity: activity_ref,
        webview: Some(webview_ref),
        files_dir,
        config,
        nav: nav::NavState {
            dialog_visible: false,
            can_go_back: false,
            has_page: false,
        },
        target_url: None,
    };

    match app::slot().lock() {
        Ok(mut guard) => *guard = Some(state),
        Err(e) => log::error!("状态锁中毒: {e}"),
    }
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnResume(
    _env: JNIEnv,
    _class: JClass,
) {
    log::debug!("onResume");
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnPause(
    _env: JNIEnv,
    _class: JClass,
) {
    log::debug!("onPause");
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnDestroy(
    _env: JNIEnv,
    _class: JClass,
) {
    log::debug!("onDestroy");
    if let Ok(mut guard) = app::slot().lock() {
        *guard = None;
    }
}
```

- [ ] **Step 5: 让 MainActivity 加载 native 库并调用 nativeCreate**

`MainActivity.kt` 完整替换为：

```kotlin
package io.github.leoyyong.codeclient

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        nativeCreate(this, filesDir.absolutePath)
    }

    override fun onResume() {
        super.onResume()
        nativeOnResume()
    }

    override fun onPause() {
        super.onPause()
        nativeOnPause()
    }

    override fun onDestroy() {
        nativeOnDestroy()
        super.onDestroy()
    }

    private external fun nativeCreate(activity: MainActivity, filesDir: String)
    private external fun nativeOnResume()
    private external fun nativeOnPause()
    private external fun nativeOnDestroy()

    companion object {
        init {
            System.loadLibrary("codeclient")
        }
    }
}
```

`System.loadLibrary("codeclient")` 会去找 `libcodeclient.so`，所以 Task 0 里 crate 的 `[lib] name = "codeclient"` 不能改。

- [ ] **Step 6: 本机验证 Rust 侧能编译**

```powershell
cargo build
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 编译通过；`cargo test` 仍是 33 个测试全绿（JNI 模块没有单测，只保证编译）。若 `AppState` 因 `GlobalRef` 不是 `Send` 而编译失败，按 Step 2 的说明处理。

- [ ] **Step 7: 提交并推送，确认 CI 绿**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 打通 Rust 与 Kotlin 的 JNI 边界并创建 WebView"
git push origin master
```

```powershell
gh run watch
```

Expected: `test` 与 `apk` 均 success。此时 APK 会打开 `https://example.com/`——足以证明 JNI 通路和 WebView 创建都成立。

---

## Task 7: 输入对话框、地址规范化与配置持久化

接通 Task 1 与 Task 2 的逻辑：冷启动读配置，无地址就弹输入框，确认后规范化并落盘。

**Files:**
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/InputDialog.kt`
- Modify: `MainActivity.kt`（加 `showInputDialog`、`nativeOnUrlSubmitted`、`nativeOnInputDialogVisibility`）
- Modify: `rust/src/lib.rs`（加导出、去掉硬编码 URL）
- Modify: `rust/src/app.rs`（加 `save_config` 辅助函数）

**Interfaces:**
- Consumes: `url::normalize`、`config::{load, save}`、`webview::load_url`
- Produces:
  - Kotlin `MainActivity.showInputDialog(prefill: String)`（Rust 经 JNI 调用）
  - Kotlin `InputDialog.show(activity, prefill, onSubmit: (String) -> Unit, onVisibilityChange: (Boolean) -> Unit)`
  - Rust `nativeOnUrlSubmitted(env, class, url: JString)`
  - Rust `nativeOnInputDialogVisibility(env, class, visible: jboolean)`
  - Rust `pub fn submit_url(env: &mut JNIEnv, input: &str) -> Result<String, UrlError>`

- [ ] **Step 1: 写输入对话框**

`InputDialog.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.app.Activity
import android.app.AlertDialog
import android.view.inputmethod.EditorInfo
import android.widget.EditText
import android.widget.FrameLayout
import androidx.core.view.setPadding

object InputDialog {

    /**
     * 显示地址输入对话框。
     *
     * 用原生对话框而不是把 WebView 导航到一个本地输入页，是为了让
     * code-server 页面在输入框弹出期间继续存活，不触发重连。
     */
    fun show(
        activity: Activity,
        prefill: String,
        errorMessage: String?,
        onSubmit: (String) -> Unit,
        onVisibilityChange: (Boolean) -> Unit,
    ) {
        val input = EditText(activity).apply {
            hint = activity.getString(R.string.input_hint)
            setText(prefill)
            setSelection(text.length)
            imeOptions = EditorInfo.IME_ACTION_GO
            maxLines = 1
        }

        val container = FrameLayout(activity).apply {
            setPadding(48)
            addView(input)
        }

        val builder = AlertDialog.Builder(activity)
            .setTitle(R.string.input_title)
            .setView(container)
            .setPositiveButton(R.string.input_confirm, null)
            .setNegativeButton(R.string.input_cancel, null)

        if (errorMessage != null) {
            builder.setMessage(errorMessage)
        }

        val dialog = builder.create()

        dialog.setOnShowListener {
            onVisibilityChange(true)
            input.requestFocus()
            dialog.window?.setSoftInputMode(
                android.view.WindowManager.LayoutParams.SOFT_INPUT_STATE_ALWAYS_VISIBLE
            )
            dialog.getButton(AlertDialog.BUTTON_POSITIVE).setOnClickListener {
                onSubmit(input.text.toString())
            }
            dialog.getButton(AlertDialog.BUTTON_NEGATIVE).setOnClickListener {
                dialog.dismiss()
            }
        }
        // 只回调一次：setOnShowListener 负责"已显示"，这里负责"已关闭"。
        dialog.setOnDismissListener { onVisibilityChange(false) }
        dialog.show()
    }
}
```

**注意 `setPositiveButton(..., null)`**：传 `null` 是为了绕开 AlertDialog 的"点击后自动关闭"行为。地址非法时要保持对话框打开并在下面显示错误，所以必须自己接管点击。这是 Android 上一个经典的坑——用 `setPositiveButton(resId) { ... }` 的简写会在校验失败时把对话框关掉。

- [ ] **Step 2: 在 MainActivity 里实现 showInputDialog**

在 `MainActivity.kt` 的类体中加入：

```kotlin
    /** Rust 经 JNI 调用。任意线程安全。 */
    fun showInputDialog(prefill: String, errorMessage: String?) {
        runOnUiThread {
            InputDialog.show(
                activity = this,
                prefill = prefill,
                errorMessage = errorMessage,
                onSubmit = { nativeOnUrlSubmitted(it) },
                onVisibilityChange = { nativeOnInputDialogVisibility(it) },
            )
        }
    }

    private external fun nativeOnUrlSubmitted(url: String)
    private external fun nativeOnInputDialogVisibility(visible: Boolean)
```

- [ ] **Step 3: 在 Rust 侧实现提交逻辑**

在 `rust/src/lib.rs` 中加入：

```rust
/// 处理输入对话框提交的地址：规范化 → 落盘 → 加载。
///
/// 返回 `Err` 时地址非法，调用方应让对话框保持打开并显示错误。
pub fn submit_url(env: &mut JNIEnv, input: &str) -> Result<String, url::UrlError> {
    let normalized = url::normalize(input)?;

    // 落盘与状态更新都在锁内完成，避免对话框连点两次时交错。
    let mut guard = match app::slot().lock() {
        Ok(g) => g,
        Err(e) => {
            log::error!("状态锁中毒: {e}");
            return Ok(normalized);
        }
    };
    let Some(state) = guard.as_mut() else {
        return Ok(normalized);
    };

    state.config.url = Some(normalized.clone());
    state.target_url = Some(normalized.clone());
    let dir = state.files_dir.clone();
    if let Err(e) = config::save(std::path::Path::new(&dir), &state.config) {
        // 配置写失败不应该阻止用户本次使用。
        log::warn!("写配置失败: {e}");
    }

    if let Some(webview) = state.webview.as_ref() {
        if let Err(e) = webview::load_url(env, &webview.as_obj(), &normalized) {
            log::error!("加载地址失败: {e}");
            let _ = env.exception_clear();
        }
    }

    Ok(normalized)
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnUrlSubmitted(
    mut env: JNIEnv,
    _class: JClass,
    url: JString,
) {
    let Some(input) = jni_env::jstring_to_string(&mut env, &url) else {
        return;
    };

    match submit_url(&mut env, &input) {
        Ok(normalized) => log::info!("已切换到 {normalized}"),
        Err(e) => {
            let message = match e {
                url::UrlError::Empty => "地址不能为空".to_string(),
                url::UrlError::UnsupportedScheme(s) => format!("不支持 {s}:// ，只能用 http 或 https"),
                url::UrlError::Malformed(d) => format!("地址无法解析：{d}"),
            };
            log::warn!("地址非法: {message}");
            // 重开对话框，保留用户输入并显示错误。
            if let Some(state) = app::with_state(|s| s.activity.clone()) {
                let Ok(activity) = env.new_local_ref(&state) else { return };
                let Ok(msg) = env.new_string(&message) else { return };
                let Ok(prefill) = env.new_string(&input) else { return };
                let _ = env.call_method(
                    &activity,
                    "showInputDialog",
                    "(Ljava/lang/String;Ljava/lang/String;)V",
                    &[
                        jni::objects::JValue::Object(&jni::objects::JObject::from(prefill)),
                        jni::objects::JValue::Object(&jni::objects::JObject::from(msg)),
                    ],
                );
                let _ = env.exception_clear();
            }
        }
    }
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnInputDialogVisibility(
    _env: JNIEnv,
    _class: JClass,
    visible: jni::sys::jboolean,
) {
    let visible = visible != 0;
    app::with_state(|s| s.nav.dialog_visible = visible);
}
```

- [ ] **Step 4: 冷启动改为读配置**

在 `nativeCreate` 中，把硬编码 URL 那段替换为：

```rust
    let mut webview_ref = webview_ref;
    let (initial_url, has_page) = match config.url.clone() {
        Some(u) => (Some(u), true),
        None => (None, false),
    };

    match &initial_url {
        Some(u) => {
            if let Err(e) = webview::load_url(&mut env, &webview_ref.as_obj(), u) {
                log::error!("加载地址失败: {e}");
                let _ = env.exception_clear();
            }
        }
        None => {
            // 没有历史地址：弹输入框，WebView 在底下保持空白。
            let Ok(activity) = env.new_local_ref(&activity_ref) else {
                log::error!("无法取得 Activity 局部引用");
                return;
            };
            let Ok(empty) = env.new_string("") else { return };
            let _ = env.call_method(
                &activity,
                "showInputDialog",
                "(Ljava/lang/String;Ljava/lang/String;)V",
                &[
                    jni::objects::JValue::Object(&jni::objects::JObject::from(empty)),
                    jni::objects::JValue::Object(&jni::objects::JObject::NULL),
                ],
            );
            let _ = env.exception_clear();
        }
    }

    // 删除原先的 const PLACEHOLDER_URL 与它的 load_url 调用。
```

并把状态构造改为：

```rust
    let state = app::AppState {
        activity: activity_ref,
        webview: Some(webview_ref),
        files_dir,
        config,
        nav: nav::NavState {
            dialog_visible: false,
            can_go_back: false,
            has_page,
        },
        target_url: initial_url,
    };
```

同时删除现在未使用的 `mut`：

```rust
    let webview_ref = webview::create(&mut env, &activity)?;
```

即 `nativeCreate` 里 `let Ok(webview_ref) = ...` 之后的 `let mut webview_ref = webview_ref;` 也不需要了，直接用 `webview_ref`。

- [ ] **Step 5: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 编译通过、clippy 无警告、33 个测试全绿。

- [ ] **Step 6: 提交并推送，确认 CI 绿**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 地址输入对话框、规范化与配置持久化"
git push origin master
```

```powershell
gh run watch
```

- [ ] **Step 7: 真机确认（可选但强烈建议）**

装 APK，确认：首次启动弹输入框；输入 `192.168.1.10:8080`（换成你的真实地址）后页面加载；杀掉应用重开，直接进入该地址且不再弹框。这一步的真机结论要记进 README 的验收清单。

---

## Task 8: 返回键状态机接入

接通 Task 4 的 `nav::decide`，并实现 Spec 7.2 的 500 ms 长按判定。

**Files:**
- Modify: `MainActivity.kt`
- Modify: `rust/src/lib.rs`

**Interfaces:**
- Consumes: `nav::{decide, NavState, BackEvent, NavAction}`、`webview::{can_go_back, go_back}`
- Produces:
  - Kotlin `MainActivity` 里的 `OnBackPressedCallback` 与长按计时
  - Rust 导出 `nativeOnBackPressed(): jboolean`（返回是否已消费）、`nativeOnBackLongPress()`、`nativeOnUiTrampoline()`
  - Rust `pub fn refresh_nav_state(env: &mut JNIEnv)`

- [ ] **Step 1: 在 Rust 侧实现返回键决策**

在 `rust/src/lib.rs` 中加入：

```rust
/// 从 WebView 实时同步导航状态里的动态字段。
fn refresh_nav_state(env: &mut JNIEnv, can_go_back: bool) {
    app::with_state(|s| s.nav.can_go_back = can_go_back);
}

fn current_can_go_back(env: &mut JNIEnv) -> bool {
    let Some(webview) = app::with_state(|s| s.webview.clone()) else {
        return false;
    };
    match env.new_local_ref(&webview) {
        Ok(local) => webview::can_go_back(env, &local),
        Err(_) => false,
    }
}

/// 执行状态机给出的动作，返回该动作是否已被消费（`false` 表示交给系统退出应用）。
pub fn apply_nav_action(env: &mut JNIEnv, action: nav::NavAction) -> bool {
    match action {
        nav::NavAction::DismissDialog => {
            // Kotlin 侧关闭对话框后会回调 nativeOnInputDialogVisibility(false)。
            if let Some(activity) = app::with_state(|s| s.activity.clone()) {
                if let Ok(local) = env.new_local_ref(&activity) {
                    jni_env::call_void(env, &local, "dismissInputDialog");
                }
            }
            true
        }
        nav::NavAction::GoBack => {
            if let Some(webview) = app::with_state(|s| s.webview.clone()) {
                if let Ok(local) = env.new_local_ref(&webview) {
                    webview::go_back(env, &local);
                }
            }
            true
        }
        nav::NavAction::ShowInput => {
            if let Some(activity) = app::with_state(|s| s.activity.clone()) {
                if let Ok(local) = env.new_local_ref(&activity) {
                    let prefill = app::with_state(|s| {
                        s.target_url.clone().or_else(|| s.config.url.clone()).unwrap_or_default()
                    })
                    .unwrap_or_default();
                    let Ok(jprefill) = env.new_string(&prefill) else { return true };
                    let _ = env.call_method(
                        &local,
                        "showInputDialog",
                        "(Ljava/lang/String;Ljava/lang/String;)V",
                        &[
                            jni::objects::JValue::Object(&jni::objects::JObject::from(jprefill)),
                            jni::objects::JValue::Object(&jni::objects::JObject::NULL),
                        ],
                    );
                    let _ = env.exception_clear();
                }
            }
            true
        }
        nav::NavAction::Finish => false,
    }
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnBackPressed(
    mut env: JNIEnv,
    _class: JClass,
) -> jni::sys::jboolean {
    let can_go_back = current_can_go_back(&mut env);
    refresh_nav_state(&mut env, can_go_back);

    let Some(state) = app::with_state(|s| s.nav) else {
        return 0;
    };
    let action = nav::decide(state, nav::BackEvent::ShortPress);
    log::debug!("短按返回: state={state:?} action={action:?}");
    u8::from(apply_nav_action(&mut env, action)) as jni::sys::jboolean
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnBackLongPress(
    mut env: JNIEnv,
    _class: JClass,
) {
    log::debug!("长按返回");
    let _ = apply_nav_action(&mut env, nav::NavAction::ShowInput);
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnUiTrampoline(
    _env: JNIEnv,
    _class: JClass,
) {
    // 目前只有加载超时计时器需要跳回 UI 线程（Task 11 使用）。
    log::debug!("UI 跳板被调用");
}
```

- [ ] **Step 2: 在 Kotlin 侧实现长按判定**

在 `MainActivity.kt` 的 `onCreate` 末尾加入：

```kotlin
        setupBackHandling()
```

并加入以下成员：

```kotlin
    private var backPressedAt = 0L
    private var longPressFired = false

    /**
     * 单个 Runnable 复用为「跳回 UI 线程」的跳板：Rust 无法方便地创建 Java 对象，
     * 所以预先造一个，Rust 只需调用 postToUi()。
     */
    private val uiTrampoline = Runnable { nativeOnUiTrampoline() }

    /** Rust 经 JNI 调用。任意线程安全。 */
    fun postToUi() {
        runOnUiThread(uiTrampoline)
    }

    /** Rust 经 JNI 调用。 */
    fun dismissInputDialog() {
        runOnUiThread { currentInputDialog?.dismiss() }
    }

    private var currentInputDialog: android.app.AlertDialog? = null

    private fun setupBackHandling() {
        onBackPressedDispatcher.addCallback(
            this,
            object : OnBackPressedCallback(true) {
                override fun handleOnBackPressed() {
                    // 长按已经在计时器里处理过了，这里只处理短按。
                    if (longPressFired) {
                        longPressFired = false
                        return
                    }
                    if (!nativeOnBackPressed()) {
                        // Rust 说该退出了。禁用本回调再手动触发，避免递归。
                        isEnabled = false
                        onBackPressedDispatcher.onBackPressed()
                    }
                }
            },
        )
    }
```

长按的判定用 `OnBackPressedCallback` 之外的路径实现会拿不到"按下"的时刻。AndroidX 的 `OnBackPressedCallback` 只在动作**完成**时回调 `handleOnBackPressed()`，因此长按要靠 `KeyEvent` 的按下/抬起时序：

```kotlin
    override fun onKeyDown(keyCode: Int, event: KeyEvent): Boolean {
        if (keyCode == KeyEvent.KEYCODE_BACK) {
            if (event.repeatCount == 0) {
                backPressedAt = SystemClock.uptimeMillis()
                longPressFired = false
                mainHandler.postDelayed(longPressCheck, BACK_LONG_PRESS_MS)
            }
            return true
        }
        return super.onKeyDown(keyCode, event)
    }

    override fun onKeyUp(keyCode: Int, event: KeyEvent): Boolean {
        if (keyCode == KeyEvent.KEYCODE_BACK) {
            mainHandler.removeCallbacks(longPressCheck)
            // 短按没到阈值就抬起：让 dispatcher 走正常短按路径。
            if (!longPressFired) {
                onBackPressedDispatcher.onBackPressed()
            }
            longPressFired = false
            return true
        }
        return super.onKeyUp(keyCode, event)
    }

    private val mainHandler = android.os.Handler(android.os.Looper.getMainLooper())

    private val longPressCheck = Runnable {
        longPressFired = true
        nativeOnBackLongPress()
    }
```

**必须同时保留 `OnBackPressedCallback` 和 `onKeyDown/onKeyUp` 两条路径**，这是 Spec 3 节里那条平台约束的直接后果：

- `android:enableOnBackInvokedCallback="true"`（Task 5 已设）下，手势返回与 3 键返回**都会**触发 `dispatcher`，但 `KEYCODE_BACK` 的按下/抬起**只在 3 键与部分 ROM 的手势下送达**。
- 因此：`onKeyDown/onKeyUp` 能收到事件时，长按由 `longPressCheck` 判定，短按由 `onKeyUp` 转交 dispatcher；收不到时（纯手势、无按键事件），由 `OnBackPressedCallback` 兜底，只是长按不可用。
- 两条路径都最终调用 `nativeOnBackPressed()` / `nativeOnBackLongPress()`，**决策仍然只在 Rust 里发生一次**。

加入导入：

```kotlin
import android.view.KeyEvent
import android.os.SystemClock
import androidx.activity.OnBackPressedCallback
```

- [ ] **Step 3: 让 InputDialog 把对话框实例回报给 MainActivity**

把 Task 7 的 `InputDialog.show` 签名改为额外返回实例：

```kotlin
    fun show(
        activity: Activity,
        prefill: String,
        errorMessage: String?,
        onSubmit: (String) -> Unit,
        onVisibilityChange: (Boolean) -> Unit,
        onCreated: (AlertDialog) -> Unit,
    ): AlertDialog {
```

在 `dialog.show()` 之前插入 `onCreated(dialog)`，并把最后一行改成 `return dialog`。

`MainActivity.showInputDialog` 相应改为：

```kotlin
    fun showInputDialog(prefill: String, errorMessage: String?) {
        runOnUiThread {
            currentInputDialog = InputDialog.show(
                activity = this,
                prefill = prefill,
                errorMessage = errorMessage,
                onSubmit = { nativeOnUrlSubmitted(it) },
                onVisibilityChange = { nativeOnInputDialogVisibility(it) },
                onCreated = { currentInputDialog = it },
            )
        }
    }
```

- [ ] **Step 4: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿。

- [ ] **Step 5: 提交并推送**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 接入返回键状态机与长按判定"
git push origin master
```

```powershell
gh run watch
```

Expected: 两个 job success。

- [ ] **Step 6: 真机验证返回键三级链**

真机上依次确认：网页能后退时短按返回会后退；退到底后短按返回弹出输入框；输入框开着时短按返回关闭它；冷启动输入框开着时短按返回退出应用；长按返回（3 键导航下按住不放）直接弹出输入框。**把每一条的实际结果记进 README 验收清单**，包括在你这台设备的手势导航下长按是否可用。

---

## Task 9: 沉浸式系统栏

**Files:**
- Create: `rust/src/ui.rs`
- Modify: `rust/src/lib.rs`（在 `nativeCreate` 里调用）
- Modify: `MainActivity.kt`（edge-to-edge）

**Interfaces:**
- Consumes: `AppState.activity`
- Produces: `pub fn apply_immersive(env: &mut JNIEnv, activity: &JObject)`

- [ ] **Step 1: 写 ui 模块**

`rust/src/ui.rs`：

```rust
use jni::objects::{JObject, JValue};
use jni::JNIEnv;

/// 隐藏状态栏，保留导航手势条。
///
/// 用 `WindowInsetsController`（API 30+）而 `WindowInsetsControllerCompat`
/// （AndroidX）是为了避免在 Rust 侧依赖 AndroidX 的类——后者在编译期不可见。
/// minSdk 是 29，所以 API 29 走 `systemUiVisibility` 回退分支。
pub fn apply_immersive(env: &mut JNIEnv, activity: &JObject) -> jni::errors::Result<()> {
    let window = env
        .call_method(activity, "getWindow", "()Landroid/view/Window;", &[])?
        .l()?;

    // edge-to-edge：内容延伸到系统栏区域之下。
    env.call_method(
        &window,
        "setDecorFitsSystemWindows",
        "(Z)V",
        &[JValue::Bool(0)],
    )?;

    let decor = env
        .call_method(&window, "getDecorView", "()Landroid/view/View;", &[])?
        .l()?;
    let insets_controller = env
        .call_method(
            &decor,
            "getWindowInsetsController",
            "()Landroid/view/WindowInsetsController;",
            &[],
        )?
        .l()?;

    // WindowInsets.Type.statusBars() == 1
    let status_bars = env
        .call_static_method(
            "android/view/WindowInsets$Type",
            "statusBars",
            "()I",
            &[],
        )?
        .i()?;

    env.call_method(
        &insets_controller,
        "hide",
        "(I)V",
        &[JValue::Int(status_bars)],
    )?;

    // BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE == 2：
    // 从屏幕边缘下滑可以临时唤出状态栏看通知。
    env.call_method(
        &insets_controller,
        "setSystemBarsBehavior",
        "(I)V",
        &[JValue::Int(2)],
    )?;

    Ok(())
}
```

`WindowInsets.Type.statusBars()` 是 API 30 的静态方法，`getWindowInsetsController` 也是 API 30。minSdk 是 29，因此在 API 29 设备上这两个调用会抛 `NoSuchMethodError`。这是可接受的：异常会被 `env.exception_clear()` 吞掉，界面退化为显示状态栏，不会崩溃。**把这个退化行为写进 README 的已知限制。**

- [ ] **Step 2: 在 nativeCreate 中调用，并清理异常**

在 `rust/src/lib.rs` 的 `nativeCreate` 里，`webview::create` 成功之后插入：

```rust
    if let Err(e) = ui::apply_immersive(&mut env, &activity) {
        // API 29 上没有 WindowInsetsController，退化为显示状态栏。
        log::warn!("设置沉浸式失败（将保留状态栏）: {e}");
        let _ = env.exception_clear();
    }
```

并加入 `pub mod ui;`。

- [ ] **Step 3: MainActivity 打开 edge-to-edge**

在 `MainActivity` 的 `onCreate` 中，`super.onCreate` 之后插入：

```kotlin
        WindowCompat.setDecorFitsSystemWindows(window, false)
```

加入导入：

```kotlin
import androidx.core.view.WindowCompat
```

- [ ] **Step 4: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿。

- [ ] **Step 5: 提交并推送**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 沉浸式系统栏"
git push origin master
```

```powershell
gh run watch
```

Expected: 两个 job success。

---

## Task 10: 证书策略接入

**Files:**
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/AppWebViewClient.kt`
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/SslPrompt.kt`
- Modify: `MainActivity.kt`（装配 client、暴露证书处理入口）
- Modify: `rust/src/lib.rs`（加 SSL 导出）

**Interfaces:**
- Consumes: `ssl::{decide, remember, SslDecision}`、`config::save`
- Produces:
  - Kotlin `AppWebViewClient(activity: MainActivity)`
  - Kotlin `SslPrompt.show(activity, host, onChoice: (allow: Boolean, remember: Boolean) -> Unit)`
  - Rust 导出：
    - `nativeSslDecision(env, class, host: JString) -> jint`（0 = 已记住允许，1 = 需要询问）
    - `nativeOnSslChoice(env, class, host: JString, allow: jboolean, remember: jboolean)`

**注意**：这里 `nativeOnSslChoice` 比 Spec 6.1 多了一个 `host` 参数。Spec 的原签名漏掉了"要记住哪个主机"这一必要信息。这个偏差是刻意的，实现完成后要回写 Spec 6.1。

- [ ] **Step 1: 写证书确认对话框**

`SslPrompt.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.app.Activity
import android.app.AlertDialog

object SslPrompt {
    fun show(activity: Activity, host: String, onChoice: (allow: Boolean, remember: Boolean) -> Unit) {
        AlertDialog.Builder(activity)
            .setTitle(R.string.ssl_title)
            .setMessage(activity.getString(R.string.ssl_message, host))
            .setCancelable(false)
            .setPositiveButton(R.string.ssl_allow_once) { _, _ -> onChoice(true, false) }
            .setNeutralButton(R.string.ssl_allow_always) { _, _ -> onChoice(true, true) }
            .setNegativeButton(R.string.ssl_cancel) { _, _ -> onChoice(false, false) }
            .show()
    }
}
```

`setCancelable(false)` 是刻意的：证书错误必须有一个明确决定，不能让用户点空白处关掉导致 `SslErrorHandler` 永远挂着——挂着的 handler 会让后续所有请求一起卡死。

- [ ] **Step 2: 写 WebViewClient（本 Task 只含 SSL 部分）**

`AppWebViewClient.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.net.http.SslError
import android.webkit.SslErrorHandler
import android.webkit.WebView
import android.webkit.WebViewClient

class AppWebViewClient(private val activity: MainActivity) : WebViewClient() {

    /**
     * 句柄留在 Kotlin（因为要等用户点对话框），策略留在 Rust。
     */
    override fun onReceivedSslError(view: WebView?, handler: SslErrorHandler?, error: SslError?) {
        if (handler == null || error == null) return

        val host = runCatching { java.net.URI(error.url).host }.getOrNull().orEmpty()

        when (nativeSslDecision(host)) {
            0 -> handler.proceed()
            else -> SslPrompt.show(activity, host) { allow, remember ->
                nativeOnSslChoice(host, allow, remember)
                if (allow) handler.proceed() else handler.cancel()
            }
        }
    }

    private external fun nativeSslDecision(host: String): Int
    private external fun nativeOnSslChoice(host: String, allow: Boolean, remember: Boolean)
}
```

- [ ] **Step 3: 在 MainActivity 里装配 WebViewClient**

Rust 创建 WebView 后不直接 `setWebViewClient`（那样 Rust 就得构造 Java 对象），而是回调 Kotlin 让它装配。在 `MainActivity` 加入：

```kotlin
    /** Rust 经 JNI 调用。必须在 UI 线程。 */
    fun attachClients(webView: android.webkit.WebView) {
        webView.webViewClient = AppWebViewClient(this)
    }
```

并在 Rust 的 `webview::create` 末尾、`setContentView` 之后插入：把新建的 WebView 作为参数回调 `attachClients`。在 `nativeCreate` 中 `webview::create` 之后插入：

```rust
    if let Some(activity) = app::with_state(|s| s.activity.clone()) {
        if let Ok(local) = env.new_local_ref(&activity) {
            if let Ok(local_wv) = env.new_local_ref(webview_ref.as_obj()) {
                let _ = env.call_method(
                    &local,
                    "attachClients",
                    "(Landroid/webkit/WebView;)V",
                    &[jni::objects::JValue::Object(&local_wv)],
                );
                let _ = env.exception_clear();
            }
        }
    }
```

**顺序问题**：`nativeCreate` 里此时 `app::slot()` 还没写入状态（状态在函数末尾才构造）。因此这段必须改成使用**局部**的 `activity_ref` 而非 `app::with_state`：

```rust
    {
        let Ok(local) = env.new_local_ref(&activity_ref) else {
            log::error!("无法取得 Activity 局部引用");
            return;
        };
        let Ok(local_wv) = env.new_local_ref(webview_ref.as_obj()) else {
            log::error!("无法取得 WebView 局部引用");
            return;
        };
        let _ = env.call_method(
            &local,
            "attachClients",
            "(Landroid/webkit/WebView;)V",
            &[jni::objects::JValue::Object(&local_wv)],
        );
        let _ = env.exception_clear();
    }
```

- [ ] **Step 4: 写 Rust 侧 SSL 导出**

在 `rust/src/lib.rs` 中加入：

```rust
#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeSslDecision(
    mut env: JNIEnv,
    _class: JClass,
    host: JString,
) -> jni::sys::jint {
    let Some(host) = jni_env::jstring_to_string(&mut env, &host) else {
        return 1;
    };
    match app::with_state(|s| ssl::decide(&s.config.ssl_allow, &host)) {
        Some(ssl::SslDecision::Allow) => 0,
        _ => 1,
    }
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeOnSslChoice(
    mut env: JNIEnv,
    _class: JClass,
    host: JString,
    allow: jni::sys::jboolean,
    remember: jni::sys::jboolean,
) {
    let Some(host) = jni_env::jstring_to_string(&mut env, &host) else {
        return;
    };
    let allow = allow != 0;
    let remember = remember != 0;
    log::info!("证书决定 host={host} allow={allow} remember={remember}");

    if allow && remember {
        let mut dir = None;
        app::with_state(|s| {
            ssl::remember(&mut s.config.ssl_allow, &host);
            dir = Some(s.files_dir.clone());
        });
        if let Some(dir) = dir {
            if let Some(cfg) = app::with_state(|s| s.config.clone()) {
                if let Err(e) = config::save(std::path::Path::new(&dir), &cfg) {
                    log::warn!("保存证书允许列表失败: {e}");
                }
            }
        }
    }
}
```

- [ ] **Step 5: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿（`ssl` 的 8 个测试仍然全绿）。

- [ ] **Step 6: 提交并推送**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 证书策略接入与按主机允许列表"
git push origin master
```

```powershell
gh run watch
```

- [ ] **Step 7: 回写 Spec**

把 Spec 6.1 里 `nativeOnSslChoice(allow: Boolean, remember: Boolean)` 改成 `nativeOnSslChoice(host: String, allow: Boolean, remember: Boolean)`，并提交：

```bash
git add docs/superpowers/specs/2026-09-23-android-vscode-web-client-design.md
git commit -m "docs: 修正 nativeOnSslChoice 签名"
```

---

## Task 11: 错误处理

**Files:**
- Create: `rust/src/error_page.rs`
- Modify: `rust/src/webview.rs`（加 `load_html`）
- Modify: `rust/src/lib.rs`（多个导出 + 超时计时器）
- Modify: `AppWebViewClient.kt`、`MainActivity.kt`
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/AppWebChromeClient.kt`

**Interfaces:**
- Consumes: `webview::load_url`、`MainActivity.postToUi()`
- Produces:
  - `error_page::render(code: i32, description: &str, url: &str) -> String`
  - `webview::load_html(env, webview, html)`（走 `loadDataWithBaseURL`）
  - Rust 导出：`nativeOnPageStarted`、`nativeOnPageFinished`、`nativeOnReceivedError`、`nativeOnReceivedHttpError`、`nativeOnRenderProcessGone`、`nativeOnProgressChanged`
  - Rust 导出：`nativeOnUiTrampoline` 现在真正干活（超时检查）
  - Kotlin `AppWebChromeClient(activity: MainActivity)`

- [ ] **Step 1: 写错误页渲染（含单测）**

`rust/src/error_page.rs`：

```rust
/// 生成内联错误页。用 data URL 加载，不依赖任何 asset 文件。
pub fn render(code: i32, description: &str, url: &str) -> String {
    format!(
        r#"<!DOCTYPE html>
<html lang="zh"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>无法加载页面</title>
<style>
 body{{font-family:system-ui,sans-serif;margin:0;padding:32px;
      display:flex;flex-direction:column;justify-content:center;
      min-height:100vh;box-sizing:border-box;
      background:#1e1e1e;color:#ddd}}
 h1{{font-size:20px;margin:0 0 12px}}
 .code{{color:#f48771;font-weight:600}}
 .url{{word-break:break-all;color:#888;font-size:13px;margin:12px 0 24px}}
 a{{display:inline-block;padding:12px 24px;background:#0e639c;color:#fff;
    text-decoration:none;border-radius:4px;align-self:flex-start}}
</style></head><body>
<h1>无法加载页面</h1>
<div><span class="code">{code}</span> {description}</div>
<div class="url">{url}</div>
<a href="codeclient://retry">重试</a>
</body></html>"#,
        code = code,
        description = html_escape(description),
        url = html_escape(url),
    )
}

fn html_escape(s: &str) -> String {
    s.replace('&', "&amp;")
        .replace('<', "&lt;")
        .replace('>', "&gt;")
        .replace('"', "&quot;")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn includes_code_and_url() {
        let html = render(-2, "net::ERR_NAME_NOT_RESOLVED", "https://a.dev/");
        assert!(html.contains("-2"));
        assert!(html.contains("net::ERR_NAME_NOT_RESOLVED"));
        assert!(html.contains("https://a.dev/"));
    }

    #[test]
    fn retry_link_uses_custom_scheme() {
        assert!(render(0, "x", "y").contains("codeclient://retry"));
    }

    #[test]
    fn escapes_html_in_description_and_url() {
        let html = render(1, "<script>alert(1)</script>", "https://a.dev/?q=<b>");
        assert!(!html.contains("<script>"));
        assert!(html.contains("&lt;script&gt;"));
        assert!(html.contains("&lt;b&gt;"));
    }
}
```

最后一条测试不是形式主义：错误描述里可能出现服务端返回的任意文本，不转义就是一个注入点。

- [ ] **Step 2: 运行测试确认失败，再实现**

```powershell
cargo test --package codeclient error_page
```

Expected（实现前）：编译失败。加上 `pub mod error_page;` 并写入上面的实现后：

Expected: `test result: ok. 3 passed`。

- [ ] **Step 3: 加 load_html**

在 `rust/src/webview.rs` 追加：

```rust
/// 用 loadDataWithBaseURL 加载内联 HTML。
///
/// baseUrl 传 null：错误页不需要相对资源，且传真实 URL 会让页面继承它的源。
pub fn load_html(env: &mut JNIEnv, webview: &JObject, html: &str) -> JniResult<()> {
    let jhtml = env.new_string(html)?;
    let null = JObject::null();
    env.call_method(
        webview,
        "loadDataWithBaseURL",
        "(Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;Ljava/lang/String;)V",
        &[
            JValue::Object(&null),                            // baseUrl
            JValue::Object(&JObject::from(jhtml)),            // data
            JValue::Object(&env.new_string("text/html")?),    // mimeType
            JValue::Object(&env.new_string("utf-8")?),        // encoding
            JValue::Object(&null),                            // historyUrl
        ],
    )?;
    Ok(())
}
```

- [ ] **Step 4: 写 WebChromeClient**

`AppWebChromeClient.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.webkit.WebChromeClient
import android.webkit.WebView

class AppWebChromeClient(private val activity: MainActivity) : WebChromeClient() {

    override fun onProgressChanged(view: WebView?, newProgress: Int) {
        nativeOnProgressChanged(newProgress)
    }

    private external fun nativeOnProgressChanged(progress: Int)
}
```

- [ ] **Step 5: 扩展 AppWebViewClient 与 attachClients**

在 `AppWebViewClient.kt` 中追加：

```kotlin
    override fun onPageStarted(view: WebView?, url: String?, favicon: android.graphics.Bitmap?) {
        nativeOnPageStarted(url.orEmpty())
    }

    override fun onPageFinished(view: WebView?, url: String?) {
        nativeOnPageFinished(url.orEmpty())
    }

    override fun onReceivedError(
        view: WebView?,
        request: android.webkit.WebResourceRequest?,
        error: android.webkit.WebResourceError?,
    ) {
        if (request?.isForMainFrame != true) return
        nativeOnReceivedError(error?.errorCode ?: -1, error?.description?.toString().orEmpty(), request.url.toString())
    }

    override fun onReceivedHttpError(
        view: WebView?,
        request: android.webkit.WebResourceRequest?,
        errorResponse: android.webkit.WebResourceResponse?,
    ) {
        if (request?.isForMainFrame != true) return
        nativeOnReceivedHttpError(errorResponse?.statusCode ?: -1, request.url.toString())
    }

    override fun onRenderProcessGone(
        view: WebView?,
        detail: android.webkit.RenderProcessGoneDetail?,
    ): Boolean {
        nativeOnRenderProcessGone(detail?.didCrash() ?: false)
        // 必须返回 true：交回给系统会让整个 Activity 被销毁。
        return true
    }

    private external fun nativeOnPageStarted(url: String)
    private external fun nativeOnPageFinished(url: String)
    private external fun nativeOnReceivedError(code: Int, description: String, url: String)
    private external fun nativeOnReceivedHttpError(statusCode: Int, url: String)
    private external fun nativeOnRenderProcessGone(didCrash: Boolean)
```

`MainActivity.attachClients` 改为：

```kotlin
    fun attachClients(webView: android.webkit.WebView) {
        webView.webViewClient = AppWebViewClient(this)
        webView.webChromeClient = AppWebChromeClient(this)
    }
```

- [ ] **Step 6: 写 Rust 侧导出与超时计时器**

在 `rust/src/lib.rs` 顶部加入：

```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::OnceLock;

/// 加载超时计时器需要跨线程告知 UI 线程"该显示错误页了"。
static LOAD_TIMEOUT_FIRED: AtomicBool = AtomicBool::new(false);
static LOAD_TIMEOUT_ARMED: AtomicBool = AtomicBool::new(false);
const LOAD_TIMEOUT_MS: u64 = 20_000;

/// 后台线程要靠它 attach 到 JVM 才能拿到 JNIEnv。
static JAVA_VM: OnceLock<jni::JavaVM> = OnceLock::new();
```

**注意**：Task 8 里已经定义过一个 `Java_..._MainActivity_nativeOnUiTrampoline` 桩函数，只打日志。下面这段是它的**替换**版本，不是新增。`#[no_mangle]` 的同名符号重复定义会导致 `duplicate symbol` 链接错误，实施时必须先删掉 Task 8 的那一版。

加入导出：

```rust
#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeOnPageStarted(
    _env: JNIEnv,
    _class: JClass,
    _url: JString,
) {
    // 页面开始加载即视为超时计时器的终点。
    LOAD_TIMEOUT_ARMED.store(false, Ordering::SeqCst);
    app::with_state(|s| {
        s.nav.has_page = true;
        s.nav.can_go_back = false;
    });
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeOnPageFinished(
    mut env: JNIEnv,
    _class: JClass,
    _url: JString,
) {
    LOAD_TIMEOUT_ARMED.store(false, Ordering::SeqCst);
    let can_go_back = current_can_go_back(&mut env);
    refresh_nav_state(&mut env, can_go_back);
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebChromeClient_nativeOnProgressChanged(
    _env: JNIEnv,
    _class: JClass,
    _progress: jni::sys::jint,
) {
    // 有进度说明连接是活的，重新计时。
    if LOAD_TIMEOUT_ARMED.load(Ordering::SeqCst) {
        let _ = arm_load_timeout();
    }
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeOnReceivedError(
    mut env: JNIEnv,
    _class: JClass,
    code: jni::sys::jint,
    description: JString,
    url: JString,
) {
    LOAD_TIMEOUT_ARMED.store(false, Ordering::SeqCst);
    let description = jni_env::jstring_to_string(&mut env, &description).unwrap_or_default();
    let url = jni_env::jstring_to_string(&mut env, &url).unwrap_or_default();
    log::warn!("页面加载失败 {code} {description} {url}");
    show_error_page(&mut env, code, &description, &url);
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeOnReceivedHttpError(
    mut env: JNIEnv,
    _class: JClass,
    status_code: jni::sys::jint,
    url: JString,
) {
    // HTTP 错误说明连接是通的，不该再报超时。
    LOAD_TIMEOUT_ARMED.store(false, Ordering::SeqCst);
    let url = jni_env::jstring_to_string(&mut env, &url).unwrap_or_default();
    log::warn!("HTTP {status_code} {url}");
    show_error_page(&mut env, status_code, "服务器返回错误状态", &url);
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeOnRenderProcessGone(
    mut env: JNIEnv,
    _class: JClass,
    did_crash: jni::sys::jboolean,
) {
    log::error!("WebView 渲染进程消失，didCrash={}", did_crash != 0);

    let Some(activity) = app::with_state(|s| s.activity.clone()) else {
        return;
    };
    let Ok(local_activity) = env.new_local_ref(&activity) else { return };

    // 旧 WebView 已经不可用，必须整个丢掉重建。
    app::with_state(|s| s.webview = None);

    let Ok(new_webview) = webview::create(&mut env, &local_activity) else {
        log::error!("重建 WebView 失败");
        let _ = env.exception_clear();
        return;
    };

    let _ = env.call_method(
        &local_activity,
        "attachClients",
        "(Landroid/webkit/WebView;)V",
        &[jni::objects::JValue::Object(new_webview.as_obj())],
    );
    let _ = env.exception_clear();

    if let Some(url) = app::with_state(|s| s.target_url.clone()) {
        let _ = webview::load_url(&mut env, new_webview.as_obj(), &url);
        let _ = env.exception_clear();
    }

    let _ = ui::apply_immersive(&mut env, &local_activity);
    let _ = env.exception_clear();

    app::with_state(|s| s.webview = Some(new_webview));
}

/// 在 WebView 里显示内联错误页。
fn show_error_page(env: &mut JNIEnv, code: i32, description: &str, url: &str) {
    let Some(webview) = app::with_state(|s| s.webview.clone()) else {
        return;
    };
    let Ok(local) = env.new_local_ref(&webview) else { return };
    let html = error_page::render(code, description, url);
    if let Err(e) = webview::load_html(env, &local, &html) {
        log::error!("加载错误页失败: {e}");
        let _ = env.exception_clear();
    }
}

/// 启动（或重启）加载超时计时器。
pub fn arm_load_timeout() -> std::io::Result<()> {
    if LOAD_TIMEOUT_ARMED.swap(true, Ordering::SeqCst) {
        return Ok(());
    }
    std::thread::Builder::new()
        .name("load-timeout".to_string())
        .spawn(|| {
            std::thread::sleep(std::time::Duration::from_millis(LOAD_TIMEOUT_MS));
            if !LOAD_TIMEOUT_ARMED.swap(false, Ordering::SeqCst) {
                return; // 已经被 onPageStarted / onProgressChanged 取消。
            }
            LOAD_TIMEOUT_FIRED.store(true, Ordering::SeqCst);
            // 跳回 UI 线程：Rust 不能直接碰 WebView。
            if let Some(activity) = app::with_state(|s| s.activity.clone()) {
                if let Ok(mut env) = attach_current_thread() {
                    if let Ok(local) = env.new_local_ref(&activity) {
                        jni_env::call_void(&mut env, &local, "postToUi");
                    }
                }
            }
        })?;
    Ok(())
}

#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_MainActivity_nativeOnUiTrampoline(
    mut env: JNIEnv,
    _class: JClass,
) {
    if LOAD_TIMEOUT_FIRED.swap(false, Ordering::SeqCst) {
        let url = app::with_state(|s| s.target_url.clone()).flatten().unwrap_or_default();
        log::warn!("加载超时 {url}");
        show_error_page(&mut env, -1, "连接超时", &url);
    }
}
```

`attach_current_thread` 是后台线程取得 `JNIEnv` 的必要手段——`JavaVM::attach_current_thread` 需要事先缓存 `JavaVM`。在 `nativeCreate` 里缓存它，并在 `nativeCreate` 中 `webview::create` 之前插入：

```rust
    if let Ok(vm) = env.get_java_vm() {
        let _ = JAVA_VM.set(vm);
    }
```

```rust
fn attach_current_thread() -> jni::errors::Result<JNIEnv<'static>> {
    let vm = JAVA_VM
        .get()
        .ok_or(jni::errors::Error::JniCall(jni::errors::JniError::Unknown))?;
    vm.attach_current_thread()
}
```

`attach_current_thread` 返回的 `JNIEnv` 在 drop 时会自动 detach，因此不能在闭包外长期持有它——上面 `arm_load_timeout` 的写法（在闭包内 attach、用完即弃）就是对的。若 jni 0.21 的实际返回类型不是 `JNIEnv<'static>`，以编译器的提示为准调整生命周期标注。

同时在 `nativeCreate` 里 `load_url` 之后调用 `arm_load_timeout()`，并加入 `pub mod error_page;`。

**为什么用轮询式的 `AtomicBool` 而不是 channel**：这只有一条"20 秒后可能触发一次"的路径，`AtomicBool` + `postToUi` 已经足够，引入 channel 只会多一个需要关闭的生命周期。注意 `arm_load_timeout` 里的 `swap(true)` 让并发重复调用只会起一个线程。

- [ ] **Step 7: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿，其中 `error_page` 3 个测试。

- [ ] **Step 8: 提交并推送**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 错误页、渲染进程崩溃恢复与加载超时"
git push origin master
```

```powershell
gh run watch
```

- [ ] **Step 9: 真机验证错误路径**

真机上把地址改成一个不存在的主机，确认显示错误页且「重试」按钮可点。这一步要真的做——错误页是唯一一条 CI 完全覆盖不到的路径。

---

## Task 12: 文件上传

**Files:**
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/FileChooser.kt`
- Modify: `AppWebChromeClient.kt`、`MainActivity.kt`

**Interfaces:**
- Consumes: `WebChromeClient.onShowFileChooser`
- Produces: `class FileChooser(activity: ComponentActivity)` 带 `launch(params: FileChooserParams, callback: ValueCallback<Array<Uri>>)` 与 `dispose()`

- [ ] **Step 1: 写文件选择管道**

`FileChooser.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.net.Uri
import android.webkit.ValueCallback
import android.webkit.WebChromeClient
import androidx.activity.ComponentActivity
import androidx.activity.result.ActivityResultLauncher
import androidx.activity.result.contract.ActivityResultContracts

/**
 * 把 WebView 的 onShowFileChooser 接到 Android 的系统文件选择器上。
 *
 * 三个 launcher 必须在这里一次性注册（ActivityResultCaller 要求在
 * Activity 创建阶段注册），不能按需注册。
 */
class FileChooser(private val activity: ComponentActivity) {

    private var pending: ValueCallback<Array<Uri>>? = null

    private val finish: (List<Uri>?) -> Unit = { uris ->
        val cb = pending
        pending = null
        // 用户取消时必须回调 null，否则下一次点击上传会静默失效。
        cb?.onReceiveValue(uris?.toTypedArray())
    }

    private val pickSingle: ActivityResultLauncher<String> =
        activity.registerForActivityResult(ActivityResultContracts.GetContent()) { uri ->
            finish(uri?.let { listOf(it) })
        }

    private val pickMultiple: ActivityResultLauncher<Array<String>> =
        activity.registerForActivityResult(ActivityResultContracts.OpenMultipleDocuments()) { uris ->
            finish(uris)
        }

    private val createDoc: ActivityResultLauncher<String> =
        activity.registerForActivityResult(ActivityResultContracts.CreateDocument("*/*")) { uri ->
            finish(uri?.let { listOf(it) })
        }

    fun launch(params: WebChromeClient.FileChooserParams, callback: ValueCallback<Array<Uri>>) {
        // 上一次还没结束就来了新请求：先把它按取消处理，避免句柄泄漏。
        pending?.onReceiveValue(null)
        pending = callback

        val accept = params.acceptTypes.filter { it.isNotBlank() }.toTypedArray()
        when (params.mode) {
            WebChromeClient.FileChooserParams.MODE_OPEN_MULTIPLE ->
                pickMultiple.launch(if (accept.isEmpty()) arrayOf("*/*") else accept)
            WebChromeClient.FileChooserParams.MODE_SAVE ->
                createDoc.launch(params.filenameHint ?: "download")
            else ->
                pickSingle.launch(accept.firstOrNull() ?: "*/*")
        }
    }

    /** Activity 销毁前调用，保证挂着的回调被结清。 */
    fun dispose() {
        pending?.onReceiveValue(null)
        pending = null
    }
}
```

- [ ] **Step 2: 接进 AppWebChromeClient**

`AppWebChromeClient.kt` 完整替换为：

```kotlin
package io.github.leoyyong.codeclient

import android.net.Uri
import android.webkit.ValueCallback
import android.webkit.WebChromeClient
import android.webkit.WebView

class AppWebChromeClient(private val activity: MainActivity) : WebChromeClient() {

    override fun onProgressChanged(view: WebView?, newProgress: Int) {
        nativeOnProgressChanged(newProgress)
    }

    override fun onShowFileChooser(
        webView: WebView?,
        filePathCallback: ValueCallback<Array<Uri>>?,
        fileChooserParams: FileChooserParams?,
    ): Boolean {
        if (filePathCallback == null || fileChooserParams == null) return false
        activity.fileChooser.launch(fileChooserParams, filePathCallback)
        return true
    }

    private external fun nativeOnProgressChanged(progress: Int)
}
```

- [ ] **Step 3: 在 MainActivity 里持有 FileChooser**

加入成员：

```kotlin
    lateinit var fileChooser: FileChooser
        private set
```

在 `onCreate` 的 `super.onCreate(...)` 之后、`nativeCreate` 之前初始化（必须在创建阶段，`registerForActivityResult` 有此时序要求）：

```kotlin
        fileChooser = FileChooser(this)
```

在 `onDestroy` 中加入：

```kotlin
        fileChooser.dispose()
```

- [ ] **Step 4: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿（本 Task 只改 Kotlin，Rust 侧无变化）。

- [ ] **Step 5: 提交并推送**

```bash
git add android/app/src/main/kotlin
git commit -m "feat: 文件上传"
git push origin master
```

```powershell
gh run watch
```

- [ ] **Step 6: 真机验证**

在 code-server 里触发一次 `<input type=file>`（例如在 Markdown 预览里插入图片），确认系统选择器弹出、选中后文件到位；再取消一次，**然后再点一次上传**——第二次必须还能弹出。第二次失效是漏调 `onReceiveValue(null)` 的典型症状。

---

## Task 13: 文件下载

**Files:**
- Create: `android/app/src/main/kotlin/io/github/leoyyong/codeclient/AppDownloadListener.kt`
- Create: `rust/src/downloads.rs`
- Modify: `MainActivity.kt`、`rust/src/lib.rs`

**Interfaces:**
- Consumes: `AppState.activity`
- Produces:
  - Kotlin `AppDownloadListener(activity: MainActivity)` 实现 `DownloadListener`
  - Rust `downloads::enqueue(env, activity, url, user_agent, content_disposition, mime_type, content_length)`
  - Rust 导出 `Java_io_github_leoyyong_codeclient_AppDownloadListener_nativeOnDownloadStart(...)`

- [ ] **Step 1: 写下载模块**

`rust/src/downloads.rs`：

```rust
use jni::objects::{JObject, JValue};
use jni::JNIEnv;

/// 解析 Content-Disposition 里的文件名。
///
/// 只处理 `filename=` / `filename*=` 两种最常见的形态；解析不出来就返回 None，
/// 由调用方退化为按 URL 末段命名。
pub fn filename_from_disposition(disposition: &str) -> Option<String> {
    let lower = disposition.to_ascii_lowercase();
    let idx = lower.find("filename=")?;
    let rest = &disposition[idx + "filename=".len()..];
    let value = rest.split(';').next()?.trim().trim_matches('"');
    let value = value.rsplit('/').next()?.rsplit('\\').next()?.trim();
    if value.is_empty() {
        None
    } else {
        Some(value.to_string())
    }
}

/// 按 URL 末段猜文件名。
pub fn filename_from_url(url: &str) -> String {
    let no_query = url.split(['?', '#']).next().unwrap_or(url);
    let last = no_query.rsplit('/').next().unwrap_or("");
    if last.is_empty() {
        "download".to_string()
    } else {
        last.to_string()
    }
}

/// 交给系统 DownloadManager。
///
/// minSdk 29 下写公共 Downloads 目录不需要任何存储权限。
pub fn enqueue(
    env: &mut JNIEnv,
    activity: &JObject,
    url: &str,
    user_agent: &str,
    content_disposition: &str,
    mime_type: &str,
) -> jni::errors::Result<()> {
    let filename = filename_from_disposition(content_disposition)
        .unwrap_or_else(|| filename_from_url(url));

    let uri = env
        .call_static_method(
            "android/net/Uri",
            "parse",
            "(Ljava/lang/String;)Landroid/net/Uri;",
            &[JValue::Object(&JObject::from(env.new_string(url)?))],
        )?
        .l()?;

    let request = env.new_object(
        "android/app/DownloadManager$Request",
        "(Landroid/net/Uri;)V",
        &[JValue::Object(&uri)],
    )?;

    env.call_method(
        &request,
        "setDestinationInExternalPublicDir",
        "(Ljava/lang/String;Ljava/lang/String;)Landroid/app/DownloadManager$Request;",
        &[
            JValue::Object(&JObject::from(env.new_string("Download")?)),
            JValue::Object(&JObject::from(env.new_string(&filename)?)),
        ],
    )?;

    // VISIBILITY_VISIBLE_NOTIFY_COMPLETED == 3
    env.call_method(
        &request,
        "setNotificationVisibility",
        "(I)Landroid/app/DownloadManager$Request;",
        &[JValue::Int(3)],
    )?;

    if !mime_type.is_empty() {
        env.call_method(
            &request,
            "setMimeType",
            "(Ljava/lang/String;)Landroid/app/DownloadManager$Request;",
            &[JValue::Object(&JObject::from(env.new_string(mime_type)?))],
        )?;
    }
    if !user_agent.is_empty() {
        env.call_method(
            &request,
            "addRequestHeader",
            "(Ljava/lang/String;Ljava/lang/String;)Landroid/app/DownloadManager$Request;",
            &[
                JValue::Object(&JObject::from(env.new_string("User-Agent")?)),
                JValue::Object(&JObject::from(env.new_string(user_agent)?)),
            ],
        )?;
    }

    let manager = env
        .call_method(
            activity,
            "getSystemService",
            "(Ljava/lang/String;)Ljava/lang/Object;",
            &[JValue::Object(&JObject::from(env.new_string("download")?))],
        )?
        .l()?;

    env.call_method(
        &manager,
        "enqueue",
        "(Landroid/app/DownloadManager$Request;)J",
        &[JValue::Object(&request)],
    )?;

    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_quoted_filename() {
        assert_eq!(
            filename_from_disposition(r#"attachment; filename="report.pdf""#).as_deref(),
            Some("report.pdf")
        );
    }

    #[test]
    fn parses_unquoted_filename() {
        assert_eq!(
            filename_from_disposition("attachment; filename=report.pdf").as_deref(),
            Some("report.pdf")
        );
    }

    #[test]
    fn strips_path_from_filename() {
        assert_eq!(
            filename_from_disposition(r#"attachment; filename="../../etc/passwd""#).as_deref(),
            Some("passwd")
        );
    }

    #[test]
    fn returns_none_without_filename() {
        assert_eq!(filename_from_disposition("attachment"), None);
        assert_eq!(filename_from_disposition(r#"filename="""#), None);
    }

    #[test]
    fn url_fallback_drops_query_and_fragment() {
        assert_eq!(filename_from_url("https://a.dev/f.zip?token=1#x"), "f.zip");
    }

    #[test]
    fn url_fallback_handles_trailing_slash() {
        assert_eq!(filename_from_url("https://a.dev/"), "download");
    }
}
```

`strips_path_from_filename` 这条测试是防目录穿越的：`Content-Disposition` 由服务端提供，`filename="../../etc/passwd"` 若不剥路径就会让 `DownloadManager` 写到 Downloads 之外。

- [ ] **Step 2: 运行测试确认失败，再通过**

```powershell
cargo test --package codeclient downloads
```

Expected（实现前）：编译失败。加上 `pub mod downloads;` 与实现后：`test result: ok. 6 passed`。

- [ ] **Step 3: 写 Kotlin 监听器**

`AppDownloadListener.kt`：

```kotlin
package io.github.leoyyong.codeclient

import android.webkit.DownloadListener

class AppDownloadListener : DownloadListener {
    override fun onDownloadStart(
        url: String?,
        userAgent: String?,
        contentDisposition: String?,
        mimeType: String?,
        contentLength: Long,
    ) {
        nativeOnDownloadStart(
            url.orEmpty(),
            userAgent.orEmpty(),
            contentDisposition.orEmpty(),
            mimeType.orEmpty(),
        )
    }

    private external fun nativeOnDownloadStart(
        url: String,
        userAgent: String,
        contentDisposition: String,
        mimeType: String,
    )
}
```

- [ ] **Step 4: 装配与 Rust 导出**

`MainActivity.attachClients` 追加：

```kotlin
        webView.setDownloadListener(AppDownloadListener())
```

`rust/src/lib.rs` 加入：

```rust
#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppDownloadListener_nativeOnDownloadStart(
    mut env: JNIEnv,
    _class: JClass,
    url: JString,
    user_agent: JString,
    content_disposition: JString,
    mime_type: JString,
) {
    let Some(url) = jni_env::jstring_to_string(&mut env, &url) else {
        return;
    };
    let user_agent = jni_env::jstring_to_string(&mut env, &user_agent).unwrap_or_default();
    let disposition = jni_env::jstring_to_string(&mut env, &content_disposition).unwrap_or_default();
    let mime_type = jni_env::jstring_to_string(&mut env, &mime_type).unwrap_or_default();

    let Some(activity) = app::with_state(|s| s.activity.clone()) else {
        return;
    };
    let Ok(local) = env.new_local_ref(&activity) else { return };

    match downloads::enqueue(&mut env, &local, &url, &user_agent, &disposition, &mime_type) {
        Ok(()) => log::info!("已交给 DownloadManager: {url}"),
        Err(e) => {
            log::error!("下载失败: {e}");
            let _ = env.exception_clear();
        }
    }
}
```

- [ ] **Step 5: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿（新增 6 个 downloads 测试）。

- [ ] **Step 6: 提交并推送**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 文件下载"
git push origin master
```

```powershell
gh run watch
```

- [ ] **Step 7: 真机验证**

从 code-server 下载一个文件，确认通知栏出现下载并落到手机的 Downloads 目录。

---

## Task 14: 外链跳系统浏览器

**Files:**
- Modify: `rust/src/url.rs`（加 `is_same_origin` + 单测）
- Create: `rust/src/intents.rs`
- Modify: `AppWebViewClient.kt`、`rust/src/lib.rs`

**Interfaces:**
- Consumes: `url::normalize`、`AppState.target_url`
- Produces:
  - `pub fn is_same_origin(a: &str, b: &str) -> bool`
  - `pub enum UrlPolicy { LoadInWebView, OpenExternal, Handled }`
  - `pub fn url_policy(target: Option<&str>, candidate: &str, has_gesture: bool) -> UrlPolicy`
  - `intents::open_external(env, activity, url)`
  - Rust 导出 `Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeUrlPolicy(url, hasGesture) -> jint`

- [ ] **Step 1: 写失败的测试**

在 `rust/src/url.rs` 的测试模块中追加：

```rust
    #[test]
    fn same_scheme_host_port_is_same_origin() {
        assert!(is_same_origin("https://a.dev/x", "https://a.dev/y?z=1"));
    }

    #[test]
    fn default_port_matches_explicit_port() {
        assert!(is_same_origin("https://a.dev/", "https://a.dev:443/"));
        assert!(is_same_origin("http://a.dev/", "http://a.dev:80/"));
    }

    #[test]
    fn different_port_is_different_origin() {
        assert!(!is_same_origin("https://a.dev/", "https://a.dev:8443/"));
    }

    #[test]
    fn different_host_or_scheme_is_different_origin() {
        assert!(!is_same_origin("https://a.dev/", "https://b.dev/"));
        assert!(!is_same_origin("https://a.dev/", "http://a.dev/"));
    }

    #[test]
    fn subdomain_is_different_origin() {
        assert!(!is_same_origin("https://a.dev/", "https://x.a.dev/"));
    }

    #[test]
    fn origin_of_garbage_is_never_same() {
        assert!(!is_same_origin("not a url", "not a url"));
    }

    #[test]
    fn retry_scheme_is_handled_internally() {
        assert_eq!(
            url_policy(Some("https://a.dev/"), "codeclient://retry", true),
            UrlPolicy::Handled
        );
    }

    #[test]
    fn gesture_on_other_host_opens_externally() {
        assert_eq!(
            url_policy(Some("https://a.dev/"), "https://github.com/x", true),
            UrlPolicy::OpenExternal
        );
    }

    #[test]
    fn redirect_without_gesture_stays_in_webview() {
        // 这是最关键的一条：code-server 的登录跳转是重定向，
        // 若按"域名不同就跳出"处理会直接打断登录。
        assert_eq!(
            url_policy(Some("https://a.dev/"), "https://login.example.com/oauth", false),
            UrlPolicy::LoadInWebView
        );
    }

    #[test]
    fn gesture_on_same_origin_stays_in_webview() {
        assert_eq!(
            url_policy(Some("https://a.dev/"), "https://a.dev/other", true),
            UrlPolicy::LoadInWebView
        );
    }

    #[test]
    fn non_http_scheme_opens_externally() {
        assert_eq!(
            url_policy(Some("https://a.dev/"), "mailto:x@y.dev", true),
            UrlPolicy::OpenExternal
        );
    }

    #[test]
    fn no_target_url_keeps_navigation_internal() {
        assert_eq!(
            url_policy(None, "https://github.com/x", true),
            UrlPolicy::LoadInWebView
        );
    }
```

- [ ] **Step 2: 运行测试确认失败**

```powershell
cargo test --package codeclient url
```

Expected: `cannot find function 'is_same_origin' in this scope`。

- [ ] **Step 3: 写实现**

在 `rust/src/url.rs` 追加：

```rust
/// 两个 URL 是否同源（scheme + host + effective port）。
pub fn is_same_origin(a: &str, b: &str) -> bool {
    let (Ok(ua), Ok(ub)) = (Url::parse(a), Url::parse(b)) else {
        return false;
    };
    ua.scheme() == ub.scheme()
        && ua.host_str() == ub.host_str()
        && ua.port_or_known_default() == ub.port_or_known_default()
}

/// 一次导航该在哪里发生。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum UrlPolicy {
    /// 让 WebView 自己加载。
    LoadInWebView,
    /// 交给系统浏览器。
    OpenExternal,
    /// Rust 已经在内部处理完了，WebView 不要加载。
    Handled,
}

/// 判定一次导航的去向。
///
/// `has_gesture` 取自 `WebResourceRequest.hasGesture()`：只有用户手指点出来的
/// 导航才算外链；重定向与 JS 跳转一律留在 WebView 内，否则 code-server 的
/// OAuth 跳转会把用户踢出应用。
pub fn url_policy(target: Option<&str>, candidate: &str, has_gesture: bool) -> UrlPolicy {
    const RETRY_SCHEME: &str = "codeclient://";
    if candidate.starts_with(RETRY_SCHEME) {
        return UrlPolicy::Handled;
    }

    let is_http = candidate.starts_with("http://") || candidate.starts_with("https://");
    if !is_http {
        // mailto: / tel: / intent: 之类交给系统处理。
        return UrlPolicy::OpenExternal;
    }

    let Some(target) = target else {
        // 还没确定目标地址，保守地留在内部。
        return UrlPolicy::LoadInWebView;
    };

    if !has_gesture || is_same_origin(target, candidate) {
        return UrlPolicy::LoadInWebView;
    }

    UrlPolicy::OpenExternal
}
```

- [ ] **Step 4: 运行测试确认通过**

```powershell
cargo test --package codeclient url
```

Expected: `test result: ok. 27 passed`（15 + 12）。

- [ ] **Step 5: 写 intents 模块**

`rust/src/intents.rs`：

```rust
use jni::objects::{JObject, JValue};
use jni::JNIEnv;

/// 用系统浏览器打开一个 URL。
pub fn open_external(env: &mut JNIEnv, activity: &JObject, url: &str) -> jni::errors::Result<()> {
    let uri = env
        .call_static_method(
            "android/net/Uri",
            "parse",
            "(Ljava/lang/String;)Landroid/net/Uri;",
            &[JValue::Object(&JObject::from(env.new_string(url)?))],
        )?
        .l()?;

    let action_view = env
        .get_static_field(
            "android/content/Intent",
            "ACTION_VIEW",
            "Ljava/lang/String;",
        )?
        .l()?;

    let intent = env.new_object(
        "android/content/Intent",
        "(Ljava/lang/String;Landroid/net/Uri;)V",
        &[JValue::Object(&action_view), JValue::Object(&uri)],
    )?;

    // 加 NEW_TASK 标志：从非 Activity 上下文启动时必需，加上无害。
    env.call_method(
        &intent,
        "addFlags",
        "(I)Landroid/content/Intent;",
        &[JValue::Int(0x1000_0000)],
    )?;

    env.call_method(
        activity,
        "startActivity",
        "(Landroid/content/Intent;)V",
        &[JValue::Object(&intent)],
    )?;

    Ok(())
}
```

- [ ] **Step 6: 接进 WebViewClient 与 Rust 导出**

`AppWebViewClient.kt` 追加：

```kotlin
    override fun shouldOverrideUrlLoading(
        view: WebView?,
        request: android.webkit.WebResourceRequest?,
    ): Boolean {
        if (request == null) return false
        return when (nativeUrlPolicy(request.url.toString(), request.hasGesture())) {
            0 -> false
            1 -> {
                activity.startActivity(
                    android.content.Intent(
                        android.content.Intent.ACTION_VIEW,
                        request.url,
                    )
                )
                true
            }
            else -> true
        }
    }

    private external fun nativeUrlPolicy(url: String, hasGesture: Boolean): Int
```

在 `rust/src/lib.rs` 加入：

```rust
#[no_mangle]
pub extern "system" fn Java_io_github_leoyyong_codeclient_AppWebViewClient_nativeUrlPolicy(
    mut env: JNIEnv,
    _class: JClass,
    url: JString,
    has_gesture: jni::sys::jboolean,
) -> jni::sys::jint {
    let Some(candidate) = jni_env::jstring_to_string(&mut env, &url) else {
        return 0;
    };
    let has_gesture = has_gesture != 0;
    let target = app::with_state(|s| s.target_url.clone()).flatten();

    match url::url_policy(target.as_deref(), &candidate, has_gesture) {
        url::UrlPolicy::LoadInWebView => 0,
        url::UrlPolicy::OpenExternal => 1,
        url::UrlPolicy::Handled => {
            // 目前只有 codeclient://retry。
            if candidate.starts_with("codeclient://retry") {
                if let Some(webview) = app::with_state(|s| s.webview.clone()) {
                    if let Ok(local) = env.new_local_ref(&webview) {
                        if let Some(retry_url) = app::with_state(|s| s.target_url.clone()).flatten() {
                            if let Err(e) = webview::load_url(&mut env, &local, &retry_url) {
                                log::error!("重试加载失败: {e}");
                                let _ = env.exception_clear();
                            }
                        }
                    }
                }
            }
            2
        }
    }
}
```

`OpenExternal` 分支里 Kotlin 自己构造 Intent，`intents::open_external` 因此在本 Task 中没有调用点。**保留 `intents.rs` 会把 clippy 的 dead_code 变成错误**（CI 用了 `-D warnings`）。两个选择：让 Kotlin 调 Rust 的 `intents`，或删掉 `intents.rs`。

选后者更简单——Kotlin 已经在那个分支里了，多绕一次 JNI 没有收益。**因此本 Task 不创建 `rust/src/intents.rs`**，`OpenExternal` 由 Kotlin 侧直接 `startActivity`。相应地 `File Structure` 表里的 `intents.rs` 作废，实施时把它从 Spec 9 节与本文档的文件表中删掉。

- [ ] **Step 7: 本机验证**

```powershell
cargo fmt --all
cargo clippy --all-targets -- -D warnings
cargo test --all
```

Expected: 全绿（url 模块 27 个测试）。

- [ ] **Step 8: 提交并推送**

```bash
git add rust/src android/app/src/main/kotlin
git commit -m "feat: 外链跳系统浏览器"
git push origin master
```

```powershell
gh run watch
```

- [ ] **Step 9: 真机验证登录路径**

这条必须真机验证：如果你的 code-server 有登录页或 OAuth，**完整走一遍登录**，确认没有被踢到浏览器。这是 Task 14 唯一可能悄悄坏掉的地方。

---

## Task 15: README 与人工验收清单

**Files:**
- Create: `README.md`
- Modify: `docs/superpowers/specs/2026-09-23-android-vscode-web-client-design.md`（删掉 `intents.rs`，回写 CI 实际可用的版本组合）

**Interfaces:**
- Consumes: 全部前序 Task
- Produces: 面向使用者的说明与可复现的验收清单

- [ ] **Step 1: 写 README**

`README.md` 至少包含以下小节，内容必须与 Spec 第 3、13 节一致：

```markdown
# Code Client

用 Rust 编写的 Android 客户端，全屏显示远端 VS Code Web 端（code-server / vscode.dev / github.dev）。

## 安装

从 GitHub Actions 的 `app-debug` artifact 下载 `app-debug.apk`，侧载安装。
需要 Android 10（API 29）或更高。

## 使用

1. 首次启动输入地址，例如 `code.example.com`、`192.168.1.10:8080`、`vscode.dev`
2. 确认后进入全屏页面
3. 短按返回：网页后退 → 回到输入框 → 退出应用
4. 长按返回：直接回到输入框

地址规范化规则见设计文档 8.5 节。

## 已知限制

- **无法在手机上「打开本地文件夹」**：VS Code Web 的 Open Folder 依赖 File System Access API，Android WebView 不支持。访问远端 code-server 的工作区不受影响。
- **Android 10 上状态栏不会被隐藏**：沉浸式依赖 API 30 的 `WindowInsetsController`，API 29 会退化为显示状态栏。
- **首次连接自签名证书的主机会弹确认框**，勾选「始终允许此主机」后不再询问。
- **长按返回在纯手势导航下可能不可用**：取决于 ROM 是否把手势返回下发为按键事件。
- **证书确认对话框限制**：读取剪贴板（粘贴）在 WebView 中支持不完整，若真机上不可用需要后续增强。

## 人工验收清单

CI 只验证编译与纯逻辑单测。以下每一条都必须在真机上手动确认：

- [ ] 首次启动弹出输入框
- [ ] 输入 `192.168.1.10:8080` 形式的地址能加载
- [ ] 输入域名不带 scheme 时按 https 加载
- [ ] 非法地址（如 `ftp://x`）显示错误且对话框不关闭
- [ ] 杀掉应用重开直接进入上次地址，不再弹框
- [ ] 短按返回：网页后退 → 输入框 → 退出，三级都对
- [ ] 长按返回直接弹输入框（记录在 3 键 / 手势导航下各自是否可用）
- [ ] 状态栏隐藏，导航手势条保留
- [ ] 软键盘弹出时不遮挡编辑区
- [ ] 屏幕旋转后 code-server 不重连
- [ ] 自签名 HTTPS 站点弹出证书确认，选「始终允许」后重开不再询问
- [ ] 点页面里的外链跳到系统浏览器
- [ ] code-server 登录流程完整走通，未被踢出应用
- [ ] 上传文件成功；取消一次后再上传仍然能弹出
- [ ] 下载文件落到手机 Downloads 目录
- [ ] 断网时显示错误页，「重试」按钮可用
- [ ] 剪贴板复制可用；粘贴是否可用（记录结论）
```

- [ ] **Step 2: 回写 Spec**

在 Spec 第 9 节的目录树里删掉 `intents`，并在第 10.1 节把工具链版本改成 CI 实际跑通的组合（AGP / Gradle / Kotlin 的确切版本号）。

```bash
git add README.md docs/superpowers/specs
git commit -m "docs: README、人工验收清单与 spec 回写"
git push origin master
```

- [ ] **Step 3: 确认最终 CI 绿且 artifact 可下载**

```powershell
gh run list --workflow=build --limit 1
gh run download --name app-debug --dir /tmp/final-apk
```

- [ ] **Step 4: 记录未通过的真机条目**

把验收清单里未通过的条目逐条记录为 GitHub Issue（或写进 README 的"已知限制"），**不要**在清单上打勾。CI 全绿不等于应用可用。

---

## 计划自检记录

**Spec 覆盖检查**（逐节对照）：

| Spec 节 | 覆盖它的 Task |
|---|---|
| 3 能力边界 | Task 5（manifest/configChanges）、Task 8（双返回路径）、Task 12/13（Kotlin 侧实现）、README |
| 4 已确认决策 | 全部 Task |
| 5 架构与线程模型 | Task 6（跳板 `postToUi`）、Task 11（后台线程 `attach_current_thread`） |
| 6.1 Kotlin→Rust 接口 | Task 6/7/8/10/11/13/14 |
| 6.2 Rust→Android 调用 | Task 6/7/9/11/13 |
| 6.3 SSL 流程 | Task 10 |
| 7.1 启动流程 | Task 7 |
| 7.2 返回键状态机 | Task 4（纯逻辑）+ Task 8（接入） |
| 7.3 显示与输入 | Task 5（manifest）+ Task 9（沉浸式） |
| 8.1 错误处理 | Task 11 |
| 8.2 文件上传 | Task 12 |
| 8.3 文件下载 | Task 13 |
| 8.4 外链 | Task 14 |
| 8.5 地址规范化 | Task 1 |
| 9 工程结构 | Task 0/5（`intents.rs` 在 Task 14 Step 6 被明确取消） |
| 10 构建与 CI | Task 5 |
| 11 测试策略 | Task 1/2/3/4/11/13/14 的单测 |
| 12 非目标 | 无 Task 需要——它定义的是"不做什么" |
| 13 真机风险 | README 验收清单（Task 15） |

**与 Spec 的已知偏差**（均为实施中发现的必要修正，Task 内已注明回写）：

1. `nativeOnSslChoice` 增加 `host` 参数（Task 10）
2. 取消 `rust/src/intents.rs`，外链由 Kotlin 直接 `startActivity`（Task 14）
3. 不提交 Gradle wrapper，CI 用 `gradle/actions/setup-gradle` 固定版本（Task 5）
4. 不声明应用图标，避免生成二进制资源（Task 5）

**类型一致性抽查**：`NavState` / `NavAction` / `BackEvent` 在 Task 4 定义、Task 8 使用，字段名一致；`UrlPolicy` 在 Task 14 定义并使用；`showInputDialog(prefill, errorMessage)` 的 `(Ljava/lang/String;Ljava/lang/String;)V` 签名在 Task 7、8、10 中一致。

