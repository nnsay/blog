---
title: AutoGLM-Phone 本地部署与实战指南
date: 2026-03-10 15:55:47
tags:
  - AI
  - DevOps
excerpt: 从零搭建 AutoGLM-Phone AI 手机助理：macOS 安卓模拟器配置、ADB 环境搭建与抖音自动化实测
---

# ChangeLog

- 2026-03-10: 初始版博客发布
- 2026-03-11: 修改模拟器配置，使用虚拟键代替模拟器边侧按键操作手机

# 声明

本文由 🤖AI 协作完成, 内容已过实际测试.

# 1. AutoGLM-Phone 简介

[AutoGLM-Phone](https://docs.bigmodel.cn/cn/guide/models/vlm/autoglm-phone) 是一个基于视觉语言模型（VLM）构建的 AI 手机智能助理框架。它能够以多模态方式深度理解屏幕内容，并通过 ADB 指令自动操控安卓设备。用户只需通过自然语言发出指令（例如“打开小红书搜索美食”），模型即可自动解析意图、识别界面元素、规划执行路径并完成操作，无需繁琐的手动交互。

# 2. 本地环境搭建

## 2.1 安卓模拟器配置

### 2.1.1 下载命令行工具

完整安装 Android Studio 虽然方便，但占用磁盘空间较大。这里推荐使用最小化命令行工具方案。请前往 [Android Studio 官网（仅限命令行工具）](https://developer.android.com/studio)下载对应工具，解压后部署到固定目录：

```bash
cd $HOME/.local/opt/
mkdir -p android_sdk/cmdline-tools
mv [cmdline-tools] android_sdk/cmdline-tools/latest
```

### 2.1.2 配置环境变量

```bash
export ANDROID_SDK_ROOT=$HOME/.local/opt/android_sdk
export PATH="$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$PATH"
export PATH="$ANDROID_SDK_ROOT/platform-tools:$PATH"
export PATH="$ANDROID_SDK_ROOT/emulator:$PATH"
```

> [!NOTE]
> `platform-tools` 和 `emulator` 将在后续步骤中自动安装，此处提前配置是为了方便后续直接调用。

### 2.1.3 下载模拟器与系统镜像

```bash
# 1. 接受许可证（此步骤为必须）
➜  ~ sdkmanager --licenses
[=======================================] 100% Computing updates...
7 of 7 SDK package licenses not accepted.
Review licenses that have not been accepted (y/N)? y

# 2. 下载模拟器核心组件
➜  ~ sdkmanager "emulator" "platform-tools"
[=======================================] 100% Unzipping... emulator/._NOTICE.tx

# 3. 下载系统镜像（推荐使用 ARM64 的 Android 12 或更高版本）
➜  ~ sdkmanager "system-images;android-33;google_apis;arm64-v8a"
[=======================================] 100% Unzipping... arm64-v8a/data/misc/
```

> [!NOTE]
>
> - 查询支持的镜像列表：`sdkmanager --list | grep "system-images"`
> - 查看适配 macOS 平台的镜像：`sdkmanager --list | grep "arm64-v8a"`
> - 若需清理磁盘空间，可删除下载的镜像：`rm -rf $ANDROID_SDK_ROOT/system-images/android-31/google_apis/x86_64`

### 2.1.4 创建并启动模拟器

````bash
# 创建一个名为 google_emulator 的模拟器
➜  ~ avdmanager create avd -n google_emulator -k "system-images;android-33;google_apis;arm64-v8a" -d "pixel_5" --force
Auto-selecting single ABI arm64-v8a     ] 25% Loading local repository...
Do you wish to create a custom hardware profile? [no]

# 可选检查: 在 ~/.android/avd/google_emulator.avd/config.ini 中，确保有如下类似的设置
```
hw.gpu.enabled=yes
```

# 查看模拟器列表
➜  ~ avdmanager list avd
Available Android Virtual Devices:
    Name: google_emulator
    Path: ~/.android/avd/google_emulator.avd
  Target: Google APIs (Google Inc.)
          Based on: Android 12.0 ("S") Tag/ABI: google_apis/arm64-v8a
  Sdcard: 512 MB

# 启动模拟器
➜  ~ emulator -avd google_emulator
INFO         | Android emulator version 36.4.9.0 (build_id 14788078) (CL:N/A)
INFO         | Graphics backend: gfxstream
INFO         | Found systemPath /Users/wangjian/.local/opt/android_sdk/system-images/android-31/google_apis/arm64-v8a/
WARNING      | Please update the emulator to one that supports the feature(s): Vulkan
INFO         | Increasing RAM size to 2048MB
````

> [!NOTE]
> 若需删除模拟器，直接删除对应的文件夹即可。通过 `avdmanager list` 可查看模拟器文件的具体存放位置。

> [!TIP]
> 查看所有支持的设备代号: avdmanager list device

### 2.1.5 测试连接

- **文件目录结构检查**

  安装完成后，`android_sdk` 目录结构应如下所示：

  ```bash
  ➜  ~ tree -L 2 .local/opt/android_sdk
  .local/opt/android_sdk
  ├── cmdline-tools
  ├── emulator
  │   ├── emulator
  ├── licenses
  ├── platform-tools
  │   ├── adb
  └── system-images
      └── android-31

  16 directories, 34 files
  ```

- **ADB 连接验证**

  测试模拟器通信是否正常：

  ```bash
  # 1. 查看已连接设备
  ➜  ~ adb devices -l
  List of devices attached
  emulator-5554          device product:sdk_gphone64_arm64 model:sdk_gphone64_arm64 device:emulator64_arm64 transport_id:1

  # 2. 在模拟器内截图并保存到内部存储
  ➜  ~ adb -s emulator-5554 shell screencap -p /sdcard/screen.png

  # 3. 将截图传回本地电脑
  ➜  ~ adb -s emulator-5554 pull /sdcard/screen.png ~/Downloads
  /sdcard/screen.png: 1 file pulled, 0 skipped. 39.9 MB/s (91544 bytes in 0.002s)
  ```

  > [!NOTE]
  > 如果模拟器内应用界面卡死，可尝试以下命令恢复：
  >
  > - 发送 Home 键指令：`adb -s emulator-5554 shell input keyevent 3`
  > - 重启桌面 Launcher：`adb -s emulator-5554 shell am start -n com.google.android.apps.nexuslauncher/.NexusLauncherActivity`

## 2.2 安装 ADB 键盘

下载并安装 `ADBKeyboard.apk`，安装完成后需进入系统设置将输入法切换为 ADB Keyboard。
（下载地址：https://github.com/senzhk/ADBKeyBoard/blob/master/ADBKeyboard.apk）

```bash
# 安装 ADB Keyboard
adb -s emulator-5554 install ADBKeyboard.apk
# 启用输入法
adb -s emulator-5554 shell ime enable com.android.adbkeyboard/.AdbIME
# 设置为默认输入法
adb -s emulator-5554 shell ime set com.android.adbkeyboard/.AdbIME
```

## 2.3 项目部署

- **克隆项目**

  ```bash
  git clone https://github.com/zai-org/Open-AutoGLM.git
  ```

- **安装依赖**

  ```bash
  uv venv --allow-existing --python 3.10 .venv
  uv pip install -r requirements.txt
  uv python pin 3.10
  ```

- **验证设备连接**

  ```bash
  adb devices
  ```

- **运行模型**

  ```bash
  uv run python main.py --base-url https://open.bigmodel.cn/api/paas/v4 --model "autoglm-phone" --apikey $AUTOGLM_API_KEY "回到首页"
  ```

# 3. 应用实测

安装目标 APK（如抖音、小红书）到模拟器：

```bash
adb -s emulator-5554 install douyin-38-0-0.apk
adb -s emulator-5554 install rednote-9-21-0.apk
```

> [!NOTE]
> 推荐访问 [Uptodown](https://cn.uptodown.com) 下载所需的 APK 安装包。

```bash
uv run python main.py --base-url https://open.bigmodel.cn/api/paas/v4 --model "autoglm-phone" --apikey $AUTOGLM_API_KEY "打开抖音,搜索美伊战争 最新消息, 注意精确定位搜索菜单位置防止误触到弹幕"
```

![image-20260310154837582](https://raw.githubusercontent.com/nnsay/gist/main/imgimage-20260310154837582.png)

# 4. QA 常见问题

## Q: 模拟器侧边控制栏反应迟钝？

- **可能原因：**
  - **焦点丢失：** 操作系统焦点未在模拟器窗口。**建议先点击一下模拟器屏幕中心**，使其获得焦点。
  - **进程同步延迟：** 直接通过 `emulator` 命令启动时，UI 面板与内核通信链路建立可能存在延迟。
  - **快捷键冲突：** macOS 系统的全局快捷键（如输入法切换）可能拦截了模拟器的按键输入。

- \*\*推荐方案：
  - **直接使用 ADB 命令替代物理按钮**

    ```bash
    # 建议添加到 .zshrc 或 .bashrc 中
    alias emu-home='adb -s emulator-5554 shell input keyevent 3'
    alias emu-back='adb -s emulator-5554 shell input keyevent 4'
    alias emu-input="adb -s emulator-5554 shell input text "
    alias emu-del="adb -s emulator-5554 shell input keyevent 67"
    ```

- **直接使用虚拟按钮**
  - 确保创建 avd 的时候设备参数: -d "pixel_5"
  - 开启虚拟三键:
    - 命令: adb shell cmd overlay enable com.android.internal.systemui.navbar.threebutton
    - 手动: 设置->System->Gestures->System navigation-> 3-button navigation
