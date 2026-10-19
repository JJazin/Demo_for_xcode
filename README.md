# LoginDemo - SwiftUI 登录页面

SwiftUI 写的登录页面 demo，深色主题，支持 email/password 登录验证。

## 本地运行

```bash
# 直接构建到模拟器
xcodebuild -project LoginDemo.xcodeproj \
  -scheme LoginDemo \
  -configuration Debug \
  -destination 'platform=iOS Simulator,name=iPhone 14' \
  build

# 安装到已开机的模拟器
xcrun simctl install booted build/Products/Debug-iphonesimulator/LoginDemo.app
xcrun simctl launch booted com.example.LoginDemo
```

**测试账号：** `test@example.com` / `password`

---

## GitHub Actions 构建 IPA 安装包

### 1. 创建 GitHub 仓库

把项目 push 到 GitHub（仓库名随意，比如 `LoginDemo`）。

### 2. 配置签名证书（Ad-hoc 方式）

Ad-hoc 可以安装到已注册的真机上：

```bash
# 生成一个自签证书（仅演示用）
# 正式项目建议用 Apple Developer Program 的正式签名
security create-keychain -p "" ~/Library/Keychains/build.keychain
security set-keychain-settings ~/Library/Keychains/build.keychain
security list-keychains -s ~/Library/Keychains/build.keychain
```

### 3. 设置 GitHub Secrets

在 GitHub 仓库 → Settings → Secrets and variables → Actions 添加：

| Secret 名 | 说明 |
|-----------|------|
| `DEV_TEAM_ID` | Apple Developer Team ID（10位字母，如 `ABCDE12345`） |

### 4. 触发构建

**方式 A：** GitHub 网页 → Actions → Build LoginDemo IPA → Run workflow

**方式 B：** 推送 tag：
```bash
git tag v1.0.0
git push origin v1.0.0
```

### 5. 下载 IPA

Actions 运行成功后，在 run 页面 Artifacts 部分下载 `LoginDemo-IPA`。

---

## 项目结构

```
LoginDemo/
├── LoginDemo.xcodeproj/
│   └── project.pbxproj
├── Sources/
│   ├── LoginDemoApp.swift      # @main 入口
│   └── LoginPage.swift          # 登录页面 UI
├── Assets.xcassets/             # App 图标
└── .github/
    └── workflows/
        └── build.yml            # CI 构建 IPA
```

## 修改 Bundle ID

在 `LoginDemo.xcodeproj/project.pbxproj` 里搜索 `com.example.LoginDemo` 替换即可。
