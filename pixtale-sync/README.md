# Pixtale Sync

照片同步的 Flutter 客户端。

## GitHub Actions 打包

工作流位于 [sync-build.yml](../.github/workflows/sync-build.yml)，固定使用 Flutter 3.47.8，与当前项目使用的 Dart 3.13.7 匹配。

1. 将工作流推送到 GitHub 默认分支。
2. 打开仓库的 **Actions → Build Pixtale Sync → Run workflow**，选择要构建的分支并运行。
3. 运行完成后，在该次运行的 **Artifacts** 下载产物：
   - `pixtale-sync-windows-x64`：包含 Windows x64 安装程序 `pixtale-setup.exe`。
   - `pixtale-sync-android-arm64`：包含 ARM64 安装包 `app-arm64-v8a-release.apk`，适用于目前大多数 Android 手机。

Android 仅构建 `arm64-v8a` 架构，不包含 ARM 32 位和 x86_64。本地可运行 `flutter build apk --release --target-platform android-arm64 --split-per-abi` 生成相同安装包。

两个平台并行构建，安装依赖后直接打包。Actions 产物保留 14 天；手动选择分支运行时仅上传 Artifacts。

### 推送标签自动发布

推送 `v*` 标签（例如 `v2.0.0`）会自动触发构建。Windows 和 Android 都成功后，工作流创建对应的 GitHub Release，标题为 `Pixtale Sync v2.0.0`，并上传：

- `pixtale-sync-v2.0.0-android-arm64.apk`
- `pixtale-sync-v2.0.0-windows-x64.exe`

Release 已存在时直接上传到该 Release，同名文件会被替换。使用内置的 `GITHUB_TOKEN` 发布，无需额外配置发布 Token。

标签必须指向包含此工作流配置的提交；此前已经推送的标签不会自动补触发。请先提交并推送工作流修改，再创建并推送新标签。发布前同步更新 `pubspec.yaml` 中的版本号和构建号，安装包内的版本仍从该文件读取。

Windows 任务安装 Inno Setup 6.5.4 及对应的简体中文语言文件，安装器版本从 `pubspec.yaml` 读取。本地仍可运行 `build.bat` 打包。

Windows 安装器启动时可选择简体中文或 English，安装向导、快捷方式选项和完成后的启动提示会使用所选语言。

### Android 签名

release APK 使用固定发布密钥签名。运行 CI 前，在仓库 **Settings → Secrets and variables → Actions** 添加以下 Repository secrets：

| Secret | 内容 |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | `.jks` 文件的 Base64 内容 |
| `ANDROID_KEYSTORE_PASSWORD` | 密钥库密码 |
| `ANDROID_KEY_ALIAS` | 密钥别名，例如 `pixtale` |
| `ANDROID_KEY_PASSWORD` | 密钥密码 |

CI 在 Runner 临时目录还原密钥，通过环境变量将签名配置传给 Gradle，构建后清理密钥。缺少任何 Secret 都会明确报错，密钥不会上传到构建产物。

本地发布密钥 `pixtale-release.jks` 和 Secrets 配置 `github-secrets.json` 放在仓库根目录的 `android-signing/` 文件夹，整个文件夹已被 Git 忽略。release 构建从 `android/key.properties` 读取配置（也已被 Git 忽略），示例：

```properties
storeFile=../../android-signing/pixtale-release.jks
storePassword=YOUR_KEYSTORE_PASSWORD
keyAlias=pixtale
keyPassword=YOUR_KEY_PASSWORD
```

`storeFile` 可以使用绝对路径；相对路径以 `android/` 为基准。Windows 路径建议使用 `/`。CI 的环境变量优先于本地配置。未配置发布签名时，release 构建会失败；debug 构建仍使用开发密钥。

密钥和密码不要提交到仓库，并安全备份。旧的 debug 签名安装包无法被新的发布签名直接覆盖，需要先卸载旧版本。
