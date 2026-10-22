# SSH Tool App

通过 Android 手机或 macOS 客户端连接 Linux 电脑，使用 Codex／Claude、查看任务结果和预览文件。macOS 也支持查看或控制远程 Linux X11 桌面。

本仓库提供编译好的安装包、被控 Linux 电脑端的飞书 P2P 安装脚本和使用教程。

## 下载安装包

| 平台 | 下载 | 版本与构建日期 |
| --- | --- | --- |
| Android | [下载 APK](https://github.com/xuhan-rgb/ssh-tool-app-releases/raw/refs/heads/main/downloads/ssh-tool-app-v1.0.3.apk) | 1.0.3，build 73，2026-10-22 |
| macOS Apple Silicon（M 系列芯片） | [下载 DMG](https://github.com/xuhan-rgb/ssh-tool-app-releases/raw/refs/heads/main/downloads/SSH_Tool_App-1.0.3-build72-macos-arm64.dmg) | 1.0.3，build 72，2026-10-22（UTC） |
| 被控 Linux x86_64／ARM64 | [下载飞书 P2P 安装脚本](https://github.com/xuhan-rgb/ssh-tool-app-releases/raw/refs/heads/main/scripts/install-feishu-p2p.sh) | 2026-10-22，单文件，内置两种架构辅助程序 |

Android 更新时直接覆盖安装，不要先卸载旧版，以保留连接配置。macOS 打开 DMG，将应用复制到“应用程序”后启动。macOS 包取自 Mac 上最新编译产物，已核对应用内版本和 build 72，并通过 DMG 完整性与签名校验。各平台功能入口以实际安装版本为准。当前未提供 Intel Mac 安装包。

以上为仓库内的直接下载，历史正式 Release 见 [版本页面](https://github.com/xuhan-rgb/ssh-tool-app-releases/releases)。App 内自动检查更新以正式 Release 为准，仓库文件更新不会自动推送到已安装 App。

可下载 [SHA256SUMS](SHA256SUMS) 核对安装包和脚本完整性；在仓库根目录执行 `sha256sum -c SHA256SUMS`。

## 普通 SSH 快速上手

1. 在被控 Linux 电脑上安装并登录 Codex／Claude，开启 SSH，并保持开机。
2. App 首页选择对应工具，点击 **新建连接**，连接方式保持 **普通 SSH**。
3. 填写电脑地址、SSH 端口、用户名和密码，保存后点击电脑卡片。
4. 首次使用 Codex 时，按提示完成 **Codex 环境配置 → 一键准备**。
5. 在 **远程打开** 中继续已有对话，或点击 **新建对话** 选择项目目录。

普通 SSH 免费，无需会员登录。客户端需能够访问电脑的 SSH 地址；局域网地址通常无法直接从外网访问。

如需自动 P2P，在电脑卡片的 **连接选项** 中开启 **优先 P2P 直连**，按需选择 **直连失败时保留原连接**。此模式仍通过可访问的 SSH 地址完成首次认证和协商。P2P 是可选付费功能，可在 **账号与授权** 登录后主动领取 7 天试用；不会自动扣款。

## 被控 Linux 安装飞书 P2P

需要 Python 3.9+、用户级 systemd，以及可访问 Python 包源和飞书的网络。将 [安装脚本](scripts/install-feishu-p2p.sh) 下载到实际使用 Codex／SSH 的 Linux 账号，以普通用户运行，不要 sudo 整个脚本：

```bash
bash install-feishu-p2p.sh
```

1. 按安装向导打开飞书授权页面，扫码创建专用应用。脚本自动保存凭据，无需手填 App ID／App Secret。
2. 按提示启用机器人，核对三项应用身份权限：`im:message.p2p_msg:readonly`、`im:message:send_as_bot`、`im:message:readonly`。使用长连接，订阅 `im.message.receive_v1` 并发布应用。
3. 在机器人私聊发送终端给出的 `/pair ...`，再发送 `/status` 验证。
4. 在支持飞书自动协商的 App 中新建连接，选择 **飞书 P2P（自动协商）**，导入电脑生成的 `client-profile.json`，或粘贴完整文件内容。
5. 点击 **获取飞书应用凭据** 完成扫码，填写 Linux SSH 用户名和密码，确认已有有效 P2P 试用或会员后保存并连接。

后续连接自动交换参数，无需人工复制飞书消息。被控电脑需运行 SSH 和 Agent；安装向导会检查服务运行、自启动和退出登录后运行的 linger 状态。飞书仅交换连接参数，SSH 数据通过 P2P 通道传输；直连失败时没有普通 SSH 地址回退。

`client-profile.json` 含设备配对密钥，请仅在自己的可信设备间传递，不要公开。它不包含 App Secret，也不能代替 SSH 认证。通用安装脚本不含个人凭据，可以单独分发。

常用命令：

```bash
bash install-feishu-p2p.sh --status
bash install-feishu-p2p.sh --logs
bash install-feishu-p2p.sh --configure
bash install-feishu-p2p.sh --reauthorize
bash install-feishu-p2p.sh --guide
```

普通重新配置会复用原授权和绑定；更换飞书账号或组织时使用 `--reauthorize`，成功后重新绑定并向控制端导入新配对文件。机器人能回复不代表两端一定可以 P2P 直连。

## 详细教程

[详细使用说明](docs/user-guide.md) 包含各平台安装、本机 Codex、SSH／飞书 P2P、环境准备、对话管理、模型与 Fast 模式、文件预览缓存和远程桌面操作。该教程描述当前源码中的功能，各安装包的可用入口以表中对应构建为准。

源码和开发文档见 [ssh-tool-app](https://github.com/xuhan-rgb/ssh-tool-app)。
