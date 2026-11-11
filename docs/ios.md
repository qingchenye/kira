# iOS 构建与自签（自用，最低 iOS 16.0）

```
push → GitHub Actions(macOS) → 未签名 ipa → 预发布 Release `ios` → 手机下载 → 自签安装
手机固定下载地址：
https://github.com/<owner>/<repo>/releases/download/ios/kira-ios-unsigned.ipa
```

构建前会自动补齐仓库里缺失的 iOS 工程（`scripts/ios_prepare.py` 在 runner 上执行，**不改仓库源码**），
所以只需要往仓库里加 2 个文件。

## 一、最快的用法：加 2 个文件

1. GitHub 仓库网页 → **Add file → Create new file**
   - 文件名：`scripts/ios_prepare.py`
   - 内容：粘贴本包 [`scripts/ios_prepare.py`](../scripts/ios_prepare.py)
   - Commit（此时还没有工作流，不会触发构建）
2. 再 **Create new file**
   - 文件名：`.github/workflows/release-ios.yaml`
   - 内容：粘贴本包 [`.github/workflows/release-ios.yaml`](../.github/workflows/release-ios.yaml)
   - Commit —— 这次提交会命中 `paths`，Actions 自动开始构建
3. （建议）Settings → Secrets and variables → Actions 加 `COPY_AUTH_KEY`：
   - 拷贝漫画 App 通道签名密钥；不配也能构建/安装，但拷贝漫画登录和接口会失败。
4. 等 Actions 里的 Build iOS IPA (unsigned) 变绿。

> 顺序：先 `scripts/ios_prepare.py`，后工作流。反过来的话第一次构建会因缺脚本报错，补上脚本后重跑一次即可（Actions → 该次运行 → Re-run）。

> 两个文件都必须加：工作流第一步会检查 `scripts/ios_prepare.py` 是否存在，缺了会直接报错退出，避免产出坏包。

## 二、构建做了什么、产物在哪

- **自动触发**：push 到 `main`/`master`，且改动含 `lib/**`、`ios/**`、`pubspec.yaml`、`pubspec.lock`、`scripts/ios_prepare.py` 或工作流自身。
- **手动触发**：Actions → Build iOS IPA (unsigned) → Run workflow：
  - `tag_name`：可选。填已存在的正式 Release 标签（如 `v1.7.6`）会把 ipa 也挂到那个 Release，应用内更新能看到；留空则只更新 `ios` 预发布。
  - `deployment_target`：默认 `16.0`；Xcode/插件要求更高时填 `17.0` 之类（本次构建生效，prepare 会同步改 Podfile 和 pbxproj）。
  - `copy_auth_key`：临时覆盖 secret。
- **构建前置检查**（任一不过直接失败，不出坏包）：
  - `ios/Podfile` 存在，`Release.xcconfig` 已接 Pods；
  - `flutter build ios` 后 `ios/Podfile.lock` 生成，且含 `flutter_inappwebview`（证明插件真被链接）；
  - ipa 内 `Info.plist` 有 `NSPhotoLibraryAddUsageDescription` / `NSPhotoLibraryUsageDescription`；
  - `lipo -info` 打印架构。
- **产物**：
  - Actions artifact `kira-ios-ipa`（30 天，含带版本号的文件名）；
  - 预发布 Release `ios` 里的 `kira-ios-unsigned.ipa`（每次构建覆盖，下载链接固定）；
  - Job Summary 里直接打印下载链接。

## 三、`scripts/ios_prepare.py` 具体改了什么

幂等：已打过补丁的仓库会直接跳过，不会重复插入。

| 目标 | 动作 |
| --- | --- |
| `ios/Podfile` | 缺失则按 Flutter 官方模板生成，`platform :ios, '16.0'`；已存在则把 platform 版本改成 16.0 |
| `ios/Flutter/Debug.xcconfig`、`Release.xcconfig` | 补 `#include? "Pods/Target Support Files/Pods-Runner/Pods-Runner.{debug,release}.xcconfig"` |
| `ios/Runner/Info.plist` | 加 `NSPhotoLibraryAddUsageDescription`、`NSPhotoLibraryUsageDescription`、`NSLocalNetworkUsageDescription`、`NSAppTransportSecurity.NSAllowsArbitraryLoads` |
| `ios/Runner.xcodeproj/project.pbxproj` | `IPHONEOS_DEPLOYMENT_TARGET` 全部改成 16.0 |
| `lib/utils/app_update.dart` | 更新源改为 `UPDATE_REPO` / `UPDATE_API_ROOT`（默认当前仓库） |
| `lib/pages/reader/reader_image_viewer.dart` | `Gal.hasAccess/requestAccess` 改为 `toAlbum: Platform.isIOS` |
| `lib/utils/display_mode_preference.dart` | 非 Android 直接返回空值，避免 `flutter_displaymode` 抛 `MissingPluginException` |

想改最低版本：工作流默认 `16.0`，可改 workflow 里 `DEPLOYMENT_TARGET` 的默认值，或本地改 `ios_prepare.py` 的 fallback。

## 四、应用内更新源

prepare 把 `app_update.dart` 改成：

```dart
static const _updateRepo = String.fromEnvironment(
  'UPDATE_REPO',
  defaultValue: 'qingchenye/kira',   // ← 想写死就改这里
);
static const _updateApiRoot = String.fromEnvironment(
  'UPDATE_API_ROOT',
  defaultValue: 'https://api.github.com',
);
```

- 工作流构建时传 `UPDATE_REPO=<当前仓库 owner/repo>`，fork 到哪查哪；
- 自建地址只要返回 GitHub Release API 兼容 JSON，用 `UPDATE_API_ROOT` 指过去；
- 稳定版检查读 `releases/latest`，自动构建的 `ios` 预发布不会被当成正式更新；想让 iOS 收到应用内提示，用 `tag_name` 把 ipa 挂到正式版本 Release；
- iOS 不能应用内装 ipa，更新卡片给的是浏览器下载链接。

## 五、自签安装

1. 手机打开固定链接下载 ipa（或从 Actions artifact 下载）；
2. ESign / Feather / Scarlet 等工具选证书 + 描述文件；
3. Bundle ID 默认 `com.example.kira`，冲突时改成 `com.qingchenye.kira` 之类；
4. 安装后到 设置 → 通用 → VPN与设备管理 信任证书；iOS 16+ 按提示开开发者模式。

## 六、可选：把补丁也落到本地仓库

有本地 kira 克隆、且这台机器能 `git push` 时：

```powershell
# 应用 + 提交 + 推送（推送后自动构建）
powershell -NoProfile -ExecutionPolicy Bypass -File .\setup.ps1 -RepoPath C:\code\kira

# 只应用，不提交
powershell -NoProfile -ExecutionPolicy Bypass -File .\apply.ps1 -RepoPath C:\code\kira
```

`apply.ps1` 会写入同一批文件并原地改 Dart/pbxproj（自动 `.bak` 备份）；之后跑工作流时 prepare 检测到已打补丁会跳过。
无论走哪条路，`scripts/ios_prepare.py` 都要提交进仓库（apply/setup 会自动加）。

## 七、已知限制（自用可接受）

- **后台下载**：iOS 没有 Android 前台服务等价物，App 切后台下载可能被挂起，建议前台下完；
- **自定义下载目录**：iOS 沙盒 + 缺安全作用域书签，重启后可能失效，建议默认目录；
- **系统字体**：`system_fonts` 偏桌面，iOS 字体列表可能为空；
- **最低 iOS 16.0**：低版本设备装不上；要放宽同时改 workflow `deployment_target` 和 Podfile/pbxproj；
- **Bundle ID** 仍是模板值，自签工具可改。

## 八、排错

| 现象 | 处理 |
| --- | --- |
| 报 `scripts/ios_prepare.py 不存在` | 两个文件都要加进仓库（路径区分大小写） |
| `pod install` 报某 pod 要求更高 iOS | 手动 Run workflow，`deployment_target` 填 `17.0` 重试 |
| 报 deployment target 不支持 | 看 prepare 步骤日志里 `IPHONEOS_DEPLOYMENT_TARGET = 16.0` 是否 3 处都改了 |
| 真机一开就退 / 大量 MissingPluginException | `Podfile.lock` 没生成或 xcconfig 没接 Pods，看构建日志里的前置检查 |
| 保存图片到相册闪退 | ipa 内 `Info.plist` 缺相册权限（工作流会校验） |
| 拷贝漫画登录失败/空白 | `COPY_AUTH_KEY` 没配或没编进去 |
| 构建没自动跑 | 提交不在 `main`/`master`，或改动没命中 `paths`；到 Actions 手动 Run |
| 下载链接 404 | 构建还没成功，先看 Actions 是否绿灯 |
| `gh release ...` 403 | 仓库 Settings → Actions → General → Workflow permissions 勾选 "Read and write permissions" |

## 九、文件清单

```
.github/workflows/release-ios.yaml   必须：自动构建 → 未签名 ipa → Release `ios`
scripts/ios_prepare.py               必须：构建前幂等补齐 iOS 工程 + 修正更新源
ios/Podfile                          可选（本地持久化补丁用）
ios/Flutter/Debug.xcconfig           可选
ios/Flutter/Release.xcconfig         可选
ios/Runner/Info.plist                可选
apply.ps1 / setup.ps1                可选：本地克隆时一键应用/提交/推送
docs/ios.md                          本文档
```
