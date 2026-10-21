# LoginDemo — SwiftUI 登录页面 + GitHub Actions 云构建

一个 SwiftUI 登录页 demo（深色主题，email/password 校验），同时演示**用 GitHub Actions 在云端编译出可安装的 IPA**。

> 本地 Xcode 14.3.1 只能用来写代码和跑模拟器；上架 App Store 需要的 Xcode 26+/iOS 26 SDK，这台 2018 年的 Mac mini 装不上 —— 所以真正的编译交给 GitHub 的 `macos-26` runner。

---

## 现状（已验证）

| 环节 | 状态 |
|---|---|
| 本地 `xcodebuild` 编译（模拟器） | ✅ 通过 |
| GitHub Actions 编译（`macos-26` runner） | ✅ 通过 |
| 产出模拟器 `.app` artifact（保留 `.app` 目录外壳） | ✅ 可下载 |
| 产出真机 arm64 `.ipa`（未签名） | ✅ 可下载 |
| 侧载安装到真机 | ⏳ 待验证（需 iPad/手机插线用 Sideloadly 签名安装） |
| 正式签名 + 上架 App Store | ⏳ 待办（需 Apple Developer Program，$99/年） |

---

## 快速上手

```bash
# 生成 Xcode 工程（工程文件由 project.yml 生成，不要手改 pbxproj）
xcodegen generate

# 编译到模拟器
xcodebuild -project LoginDemo.xcodeproj \
  -scheme LoginDemo \
  -configuration Debug \
  -destination 'platform=iOS Simulator,name=iPhone 14' \
  build

# 安装并启动到已开机的模拟器
xcrun simctl install booted <derivedData>/Build/Products/Debug-iphonesimulator/LoginDemo.app
xcrun simctl launch booted dev.jjazin.LoginDemo
```

**测试账号：** `admin` / `123456`

---

## 项目结构

```
LoginDemo/
├── project.yml                 # ★ 唯一真源：XcodeGen 工程描述
├── LoginDemo.xcodeproj/        # 由 xcodegen generate 生成，勿手改
├── Sources/
│   ├── LoginDemoApp.swift      # @main 入口
│   └── LoginPage.swift         # 登录页面 UI
├── Assets.xcassets/            # App 图标 + AccentColor
├── .github/
│   └── workflows/
│       └── build.yml           # CI：构建 .app + .ipa
└── README.md
```

### 为什么用 XcodeGen

手写的 `project.pbxproj` 在 Xcode 26 上会让 `xcodebuild` **Segfault**（exit 139），且报错信息毫无指向性。改用 `project.yml` 描述工程、`xcodegen generate` 生成标准 pbxproj 后，CI 稳定通过。

`project.yml` 里还声明了 `schemes:` 段，XcodeGen 会一并生成共享 scheme —— 这是之前 "Scheme is not currently configured for the build action" 报错的解法。

改工程配置的正确姿势：**改 `project.yml` → 跑 `xcodegen generate` → 提交两个文件**。

---

## 云端构建（GitHub Actions）

### 触发

Actions 页��� → **Build LoginDemo (Free, no signing)** → **Run workflow**。

也可以用 API：

```bash
curl -X POST "https://api.github.com/repos/<owner>/<repo>/actions/workflows/build.yml/dispatches" \
  -H "Authorization: token $TOKEN" -H "Content-Type: application/json" \
  -d '{"ref":"main"}'
```

### 产物

| Artifact | 内容 | 用途 |
|---|---|---|
| `LoginDemo-App-Simulator` | 解压后是 `LoginDemo.app/` 目录（x86_64 + arm64） | 装进 Xcode 模拟器 |
| `LoginDemo-IPA-Unsigned` | `LoginDemo-unsigned.ipa`（真机 arm64，未签名） | 用 Sideloadly / AltStore 侧载到真机 |

最新验证运行：[`39553424116`](https://github.com/JJazin/Demo_for_xcode/actions/runs/39553424116)（commit `8dbd1f7`，success）。

### 四个必须注意的坑

**1. PAT 需要 `workflow` 权限**（经典 PAT）或多一项 `Workflows: Read and write`（细粒度 PAT）。
GitHub 会对 `.github/workflows/` 下的文件做**服务端强制拦截** —— 缺权限时，Contents API、Git Data API（tree/commit）、`git push` 全部失败。注意 `Contents: write` 单独**不够**，必须单独授权 `Workflows`。

**2. 产物必须落在 workspace 内。**
`xcodebuild` 默认写到 `~/Library/Developer/Xcode/DerivedData/`，而 `upload-artifact` 只在 workspace 里搜 —— 匹配不到时 v4 **默认静默跳过**，整个 run 依然显示绿色 success（假绿）。修法：

```yaml
- derivedDataPath dd
```
并把路径写成 `dd/Build/Products/Debug-iphonesimulator/LoginDemo.app`，同时加 `if-no-files-found: error` 让它以后直接报错而不是假装成功。

**3. 模拟器 destination 不要写死机型。**
写 `name=iPhone 16` 在 runner 上会报 `Unable to find a device matching the provided destination specifier`（runner 上没有命名的模拟器设备）。用 `generic/platform=iOS Simulator`。

**4. `upload-artifact` 会压平顶层目录。**
上传单个路径 `out-sim/LoginDemo.app` 时，artifact 里只剩 `.app` 的**内容**（`LoginDemo`、`Info.plist`…），`.app` 外壳丢失，下载后无法直接安装。修法：把 `.app` 放进一个空的父目录，上传**父目录**（本仓库用 `path: "out-sim"`），artifact 根目录里就会保留 `LoginDemo.app/`。

---

## 侧载到真机（免费方案）

CI 产出的是**未签名** IPA —— iOS 要求所有非 App Store 应用必须有有效签名。用 **Sideloadly**（https://sideloadly.io）现场签名：

1. 数据线连接设备，手机上「信任此电脑」
2. 把 `.ipa` 拖进 Sideloadly，填 Apple ID 和密码
3. 点 **Start**
4. 设备上：设置 → 通用 → VPN与设备管理 → 信任你的 Apple ID

⚠️ 免费 Apple ID 签的应用**只有 7 天有效期**，到期重新装一次；同一账号最多侧载 3 个 App。

macOS 上 Sideloadly 不需要 iTunes（内置 libimobiledevice，走系统自带 usbmuxd）。

---

## 上架 App Store（待办）

等拿到 Apple Developer Program 账号后，在现有流水线上补三步即可，工程结构不用动：

1. `DEVELOPMENT_TEAM` + `CODE_SIGN_STYLE = Automatic`
2. `-destination 'generic/platform=iOS'` 配合 `archive` + `exportArchive` 导出正式签名的 IPA
3. 用 App Store Connect API Key（P8）把 IPA 上传上去

---

## 改 Bundle ID

改 `project.yml` 里的 `PRODUCT_BUNDLE_IDENTIFIER`，然后 `xcodegen generate`。
