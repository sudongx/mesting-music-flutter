# macOS UI 协同开发

## 开发基线

2026-11-06 从 `beta` 的 `f28f57ecd27bfa4441e041553a86028abdb6422d` 建立
`codex/ui-polish`。当前产品为 `1.0.60+81`。接续时先阅读根目录
`AGENTS.md` 与 `HANDOFF-CURRENT-SESSION-2026-11-05-v1.0.60.md`。

工具链采用 Flutter 3.41.9、Dart 3.11.5、JDK 21、Android SDK Platform 36、
Build Tools 36.0.0 和 NDK 28.2.13676358。Node.js 22 用于既有 JavaScript
桥接回归，GitHub CLI 用于认证、推送权限检查和 PR。

Flutter 框架提交应为 `00b0c91f06209d9e4a41f71b7a512d6eb3b9c694`，
引擎 revision 应为 `42d3d75a56efe1a2e9902f52dc8006099c45d937`。
Beta Release 构建消费仓库内的双 ARM 定制引擎及完整许可，Gradle 会检查摘要。

## 本机环境

本次工具安装在 `$HOME/.local/share/mesting-toolchain`。新终端使用下列命令启用
项目工具。`env.sh` 是本机生成的配置文件，路径可按接手机器调整。

```sh
source "$HOME/.local/share/mesting-toolchain/env.sh"
flutter --version
java -version
node --version
gh --version
flutter doctor -v
adb devices -l
# 启动 DevTools 后，可连接调试构建的 VM Service。
dart devtools --no-launch-browser
```

`android/local.properties` 在本机填写 `sdk.dir` 和 `flutter.sdk`，分别指向
该目录下的 `android-sdk` 与 `flutter-3.41.9`。签名配置使用本机私有文件。
依赖恢复使用锁定模式。

```sh
flutter pub get --enforce-lockfile
dart analyze lib test
```

## 基线回归

本轮原始日志保存于 `.codex-artifacts/ui-prep-2026-11-06/`。
本轮执行结果见 `validation-ui-prep-2026-11-06.json`。

Beta 文件清单取自 1.0.60 验证记录，转换其中的历史 Windows 路径后运行。

```sh
python3 - <<'PY'
import json
import subprocess

with open('docs/validation-2026-11-05-v1.0.60.json') as source:
    recorded = json.load(source)
files = ['test/' + name.rsplit('/', 1)[-1]
         for name in recorded['host']['betaPerFile']]
subprocess.run(['flutter', 'test', '--no-pub', '--flavor', 'beta',
                '--reporter', 'json', *files], check=True)
PY
```

Stable 与导航回归独立执行，保留各组完整日志及退出状态。

```sh
flutter test --no-pub --flavor stable \
  test/app_update_test.dart test/beta17_player_ui_test.dart \
  test/chat_page_test.dart test/csnas_app_update_test.dart \
  test/music_search_ui_test.dart test/playlist_detail_ui_test.dart
flutter test --no-pub --flavor beta test/beta5_ui_test.dart
```

本机首次完整回归出现 Node.js 缺失，补装后桥接测试通过。长期 TLS fixture
在本机校验失败，macOS 测试现使用系统 OpenSSL 与公开测试密钥，在临时目录
生成 365 天有效的证书并在测试后清理。单文件 8 项及完整 Beta 回归 2291 项均已通过，1 项条件跳过。

本轮本机 `flutter build apk --debug --flavor beta` 构建通过，产物为
`build/app/outputs/flutter-apk/app-beta-debug.apk`。构建过程按插件要求补装
SDK 33–35、Build Tools 35 和 CMake 3.22.1。

## 真机与 UI 验收

本次用户指定已连接真机首次安装最新 Release。下载源为 GitHub 的
`v1.0.60` Release，安装包 `Mesting-Music-v1.0.60.apk`。

- 包名为 `com.mesting.music`，版本为 `1.0.60`，Android code 为 81。
- APK SHA-256 为 `f1adfd1d1c6ec2aec71489ea720328ca629d713a1b032de849c802de0d082f58`。
- 签名证书 SHA-256 为 `8ce2203ad821ad8d38576f9ae86350d98254ac05f2cb974e773c1973b22e0333`。

安装前通过 `apksigner verify --print-certs` 校验证书，通过 SHA-256 校验下载。

用户确认本分支负责 UI 开发，发行由原仓库主人合并代码后打包。Beta Debug
采用 `com.mesting.music.dev` 与本机调试签名，可独立安装并通过 Flutter 热重载。
在项目根目录执行下列命令，设备标识取自 `adb devices -l`。

```sh
flutter run --debug --flavor beta -d DEVICE_SERIAL \
  --dart-define=USE_CLOUDBASE_AUTH=true \
  --dart-define=ENABLE_APP_UPDATE_CHECKS=false
```

本轮已验证调试包安装、启动、DevTools 连接及热重载。首次使用调试包时，
在手机上完成欢迎页的协议确认及游客体验或登录。

UI 验收逐项采集首页、播放器、搜索和个人页的浅色与深色对照，检查窄屏、
大字体、键盘、安全区、滚动和播放状态切换。玻璃按钮遵循
`BETA-CONTROL-CONVENTIONS.md`，检查从按钮起手拖动时的按压恢复。
帧耗时采样使用项目现有隔离性能包，遵循 `AGENTS.md` 的设备约定。
截图和设备日志保存在本机，分享前检查账号资料。

## GitHub 协同

当前登录账号为 `sudongx`，对原仓库拥有 READ 权限。本机保留 `origin` 指向
原仓库，增加 `fork` 指向 `sudongx/mesting-music-flutter`，已通过 Fork 提交
[准备资料 PR #2](https://github.com/Mesting42/mesting-music-flutter/pull/2)。

每项 UI 修改单独提交。验证完成后同步原仓库的 `beta`，处理并发改动并重跑相关
回归，然后向 `fork` 推送 `codex/ui-polish`，更新以 `beta` 为目标分支的 PR。
PR 记录修改范围、前后截图、实际测试结果和待验项目，推送后回读远端 SHA。

```sh
gh auth status
git fetch origin beta
git merge origin/beta
git push -u fork codex/ui-polish
# 创建新的 PR 时明确原仓库与 Fork 来源。
gh pr create --repo Mesting42/mesting-music-flutter \
  --base beta --head sudongx:codex/ui-polish
```

本轮按环境准备任务提交接续资料。具体页面的 UI 修改在用户提供目标后逐项开展。
