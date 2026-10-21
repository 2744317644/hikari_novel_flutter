# 编译 iOS 版（未签名 ipa）

## 前提：iOS 包不能在 Windows 上编译

苹果的代码签名和 Xcode 工具链只跑在 macOS 上。Windows 上既没有 `xcodebuild`，
也没有 iOS SDK，`flutter build ios` 会直接报
`Building for iOS is only supported on macOS`。所以「本机一键出 ipa」这条路不存在，
只能在 macOS 上构建 —— 要么借 GitHub Actions 的 macOS 运行器（免费、无需自备 Mac），
要么用你自己的 Mac。

本仓库的 `.github/workflows/build-ios.yml` 走的是第一条路。

---

## 路线 A：用 GitHub Actions 出未签名 ipa（推荐）

### 1. Fork 仓库

打开 <https://github.com/15dd/hikari_novel_flutter>，点右上角 **Fork**，
创建到自己的账号下（保持 public，macOS 运行器对公开仓库免费）。

### 2. 启用 Actions

Fork 出来的仓库默认不会自动跑 workflow。进入自己仓库的 **Actions** 标签页，
如果看到 「Workflows aren't being run on this forked repository」，
点 **I understand my workflows, go ahead and enable them**。

### 3. 把本 workflow 推到 fork 的 main 分支

本仓库已经把这两个文件加好了，随代码一起 push 即可：

```
.github/workflows/build-ios.yml
ios/Runner/Info.plist   # 补了相册权限说明等 iOS 必需项
```

推到 `main` 之后会**自动触发一次构建**（因为 workflow 里有 `on.push`），
也可以在 Actions 页面里手动点 **Run workflow**。

### 4. 等构建、下载 ipa

首次构建约 15–25 分钟（要拉 Flutter SDK、装 CocoaPods 依赖）。
成功后到这次运行页面底部 **Artifacts** 下载 `HikariNovel-unsigned-ipa`，
解压得到类似 `HikariNovel-0.5.0-beta.1-unsigned-12.ipa`。

失败的话，同页面有 `ios-build-log` 产物，里面是完整的 `flutter build ios` 日志。

### 5. 装到 iPhone

ipa 是**未签名**的，需要自己签：

| 工具 | 说明 |
|------|------|
| **Sideloadly**（Windows 可用） | 数据线连 iPhone，填自己的 Apple ID 签名安装。免费账号签的 app **7 天后过期**，重新签一次即可 |
| **AltStore / SideStore** | 手机上自签，同样受 7 天限制，但可以在手机上续签 |
| **爱思助手** | 国内常见，可直接导入 ipa 安装（依赖它自己的签名服务） |
| **TrollStore** | 仅限支持的系统版本，装完永久有效，不需要 7 天重签 |

注意：
- 免费 Apple ID 签名要在 iPhone 上先开启「设置 → 隐私与安全性 → 开发者模式」。
- 未签名 ipa 的 Bundle ID 是 `com.example.hikariNovelFlutter`，
  侧载工具重签名时通常会自动改写；如果想要固定 ID，把
  `ios/Runner.xcodeproj/project.pbxproj` 里的 `PRODUCT_BUNDLE_IDENTIFIER`
  改成自己的（例如 `com.yourname.hikari`）再构建。

---

## 路线 B：自己有 Mac

在 Mac 上（需要完整安装 Xcode，并把 Xcode 至少启动一次接受协议）：

```bash
# 只需装 Command Line Tools 是不够的，必须装完整 Xcode
sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch

git clone https://github.com/<你的账号>/hikari_novel_flutter.git
cd hikari_novel_flutter
flutter pub get

# 未签名 ipa（和 CI 产物一样）
flutter build ios --release --no-codesign
mkdir -p build/ipa/Payload
cp -R build/ios/iphoneos/Runner.app build/ipa/Payload/
cd build/ipa && zip -qry ../HikariNovel.ipa Payload
```

想直接装到自己手机，用 `open ios/Runner.xcworkspace` 打开 Xcode，
在 Runner target 的 **Signing & Capabilities** 里选自己的 Team（Add Account 加 Apple ID），
然后连手机 Run，或用 `flutter run --release -d <设备id>`。

---

## 路线 C：上架 / TestFlight

需要 Apple Developer Program（$99/年）。拿到开发者账号后：

1. 改 Bundle ID 为你在 App Store Connect 注册的标识（不能是 `com.example.*`）。
2. 在本地 Xcode 里配好 Team 和签名，或准备证书与描述文件。
3. `flutter build ipa --release` 产出 `build/ios/ipa/*.ipa`。
4. 用 Transporter 或 `xcrun altool` 上传。

如果想让 CI 也带签名，需要在仓库 Secrets 里放证书（base64 的 `.p12`）、
描述文件、Team ID、App Store Connect API Key，并在 workflow 里解出证书、装描述文件后
去掉 `--no-codesign`。这部分等有账号了再加。

---

## 关于 Xcode 和 Flutter 版本

**Xcode 必须用 26.x。** `macos-15` 镜像默认选中的是 Xcode 16.4，而
`device_info_plus` 用到了新 SDK 才有的 `NSProcessInfo.isiOSAppOnVision`，
在 16.4 下会直接编译失败：

```
ARC Semantic Issue (Xcode): No visible @interface for 'NSProcessInfo'
declares the selector 'isiOSAppOnVision'
```

镜像是装了 26.x 的（`/Applications/Xcode_26.0 ~ 26.3.app`），workflow 里的
`Select newest Xcode` 步骤会自动挑最新的那个并 `xcode-select` 过去。若哪天镜像变了，
这一步也会打印实际使用的版本，方便排查。

`pubspec.lock` 里锁定的 `charset_converter 2.5.1` 要求 **Flutter ≥ 3.44.0**
（`charset_converter` 负责 GBK/UTF-8 转码，缺了它小说正文会乱码）。
workflow 用 `channel: stable`，即当前最新稳定版，天然满足；
如果哪天换成固定版本，别低于 3.44.0。

---

## 常见失败点

| 现象 | 原因与处理 |
|------|-----------|
| `requires a higher Dart SDK` / `^3.12.2` 不满足 | runner 上的 stable 比项目要求旧。在 workflow 的 Setup Flutter 步骤加上 `flutter-version: 'x.y.z'` 固定版本 |
| `pub get` 超时或 404 | `pubspec.lock` 记录的是 `pub.flutter-io.cn` 镜像，workflow 已用 `PUB_HOSTED_URL` 对齐；连续失败 3 次会自动回退到官方 `pub.dev` 再试 |
| `CocoaPods could not find compatible versions` | 某个插件要求 iOS 版本高于 `IPHONEOS_DEPLOYMENT_TARGET`（当前 13.0）。按日志把 pbxproj 里的目标调高即可 |
| `No such module 'flutter_inappwebview_ios'` | `pod install` 没跑完，通常是网络问题，重跑一次即可 |
| 启动即崩溃、日志提示缺少 `NSPhotoLibraryUsageDescription` | 已在本仓库的 `ios/Runner/Info.plist` 补齐，若你 fork 了旧版本需同步该改动 |

---

## 本次改动清单

- 新增 `.github/workflows/build-ios.yml`：macOS 运行器构建未签名 ipa 并上传产物。
- 修改 `ios/Runner/Info.plist`：补 `NSPhotoLibraryUsageDescription`、
  `NSPhotoLibraryAddUsageDescription`、`ITSAppUsesNonExemptEncryption`、
  `CFBundleLocalizations`。
- 新增本文件 `docs/build-ios.md`。

## 把改动推到你自己的 fork

```bash
git config user.name "你的名字"      # 若本机还没配过
git config user.email "你的邮箱"

git add .github docs ios/Runner/Info.plist
git commit -m "ci: 添加 iOS 未签名 ipa 构建工作流"

# origin 是上游仓库，没有推送权限，所以推到自己的 fork
git remote add fork https://github.com/<你的账号>/hikari_novel_flutter.git
git push fork main
```

不想用命令行的话，也可以在 GitHub 网页上 **Add file → Upload files**，
把 `.github/workflows/build-ios.yml`、`docs/build-ios.md` 传上去，
并在线编辑 `ios/Runner/Info.plist` 补上那几项。
