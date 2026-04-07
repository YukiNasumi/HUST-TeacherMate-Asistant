# yatm - 微助教自动签到工具

> Yet Another TeacherMate Helper

## 项目简介

yatm 是一个微助教（TeacherMate）平台的自动签到工具，支持三种签到模式：普通签到、GPS 定位签到和二维码签到。

## 原理

### 核心思路

微助教的后端通过微信 OAuth 生成的 `openId` 来识别学生身份，但**不校验请求是否来自微信客户端**。因此只要拿到有效的 `openId` 并伪造微信的 `User-Agent`，就可以直接调用签到 API 完成签到。

### 三种签到类型

**普通签到 & GPS 签到**（全自动）

程序每隔 `interval` 毫秒轮询微助教服务器，查询当前是否有活跃的签到活动。检测到后等待 `wait` 毫秒（避免秒签太可疑），然后向签到接口发一个 POST 请求即可完成。GPS 签到会额外附带配置文件中预设的经纬度坐标。

**二维码签到**（半自动）

老师端的二维码每隔几秒会刷新，每个二维码对应不同的签到 URL。程序通过 WebSocket（Faye 协议）订阅二维码更新频道，实时获取最新的签到 URL，并在终端打印对应的二维码。你需要用微信手动扫码完成签到。

### openId 的有效期

`openId` 会在以下情况失效：
- 发送了大量请求后过期
- 从微信中重新进入微助教页面（触发新的 OAuth 授权）
- 完成一次二维码扫码签到后

过期后需要重新获取。

## 环境要求

- [Node.js](https://nodejs.org/) >= 14
- 包管理器：[pnpm](https://pnpm.io/)（推荐）、yarn 或 npm
- Git（可选，用于克隆仓库）

### 安装 Node.js

从 [Node.js 官网](https://nodejs.org/) 下载安装，或使用版本管理工具：

```bash
# macOS (Homebrew)
brew install node

# 或使用 nvm（推荐，方便管理多版本）
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
nvm install 18
```

### 安装 pnpm

```bash
npm install -g pnpm
```

## 构建

```bash
# 安装依赖
pnpm install

# 编译 TypeScript
pnpm build
```

## 配置

### 创建配置文件

```bash
cp sample.config.json config.json
```

### 配置项说明

```json
{
  "interval": 10000,
  "wait": 5000,
  "lat": 30.511227,
  "lon": 114.41021,
  "clipboard": {
    "paste": "pbpaste",
    "copy": "echo {} | pbcopy"
  },
  "qr": {
    "mode": "terminal"
  },
  "ua": ""
}
```

| 配置项 | 类型 | 说明 |
|--------|------|------|
| `interval` | number | 签到检测轮询间隔（毫秒）。程序每隔这个时间查询一次服务器是否有活跃签到 |
| `wait` | number | 检测到普通/GPS签到后等待的时间（毫秒），避免秒签被老师注意 |
| `lat` | number | GPS 签到使用的纬度坐标 |
| `lon` | number | GPS 签到使用的经度坐标 |
| `clipboard.paste` | string | 读取剪贴板的系统命令（详见下方剪贴板配置） |
| `clipboard.copy` | string | 写入剪贴板的命令，`{}` 是内容占位符（详见下方剪贴板配置） |
| `qr.mode` | string | 二维码签到的展示方式：`terminal`（终端打印二维码）、`plain`（打印 URL）、`copy`（打印 URL 并复制到剪贴板） |
| `ua` | string | 微信 User-Agent。建议填自己的，留空则使用默认 UA |

### 剪贴板配置

配置 `clipboard` 后，程序会自动从系统剪贴板轮询读取 openId，无需手动粘贴到终端。不同操作系统的配置如下：

**macOS**

macOS 自带 `pbpaste` 和 `pbcopy`，无需额外安装：

```json
"clipboard": {
  "paste": "pbpaste",
  "copy": "echo {} | pbcopy"
}
```

**Linux（X11）**

需要先安装 `xclip`：

```bash
# Debian/Ubuntu
sudo apt install xclip

# Arch Linux
sudo pacman -S xclip
```

```json
"clipboard": {
  "paste": "xclip -selection clipboard -o",
  "copy": "echo {} | xclip -selection clipboard"
}
```

如果使用 Wayland 而非 X11，改用 `wl-clipboard`：

```bash
sudo apt install wl-clipboard
```

```json
"clipboard": {
  "paste": "wl-paste",
  "copy": "echo {} | wl-copy"
}
```

**Windows（PowerShell）**

Windows 自带 `clip` 和 PowerShell 的 `Get-Clipboard`：

```json
"clipboard": {
  "paste": "powershell -command Get-Clipboard",
  "copy": "echo {} | clip"
}
```

**不使用剪贴板**

如果不配置 `clipboard` 字段，程序启动时会在终端提示手动输入 openId，或通过环境变量 `OPEN_ID` 传入。

### UA（User-Agent）获取方式

`ua` 是微信内置浏览器的 User-Agent 字符串，用于伪装请求来源。获取方法：

1. 在微信中打开 `https://www.whatismybrowser.com/detect/what-is-my-user-agent/`
2. 页面会显示你的 UA，复制填入配置即可

留空则使用内置的默认 UA，也能正常工作。

## 使用

### 1. 获取 openId

在微信中打开「微助教服务号」，点击 **学生(S)** → **签到**，然后复制页面 URL。URL 中包含 `openId` 参数。



详见 [获取 openId 教程](./docs/AcquireOpenID.md)。

### 2. 启动程序

**方式一：剪贴板模式**（推荐，需配置 `clipboard`）

```bash
pnpm start
```

启动后将包含 openId 的链接复制到系统剪贴板，程序会自动读取。

**方式二：环境变量传入**

```bash
# macOS / Linux
env OPEN_ID=你的openId pnpm start

# Windows (PowerShell)
$env:OPEN_ID="你的openId"; pnpm start

# Windows (CMD)
set OPEN_ID=你的openId && pnpm start
```

**方式三：手动输入**

如果未配置 `clipboard` 且未设置环境变量，程序启动后会在终端提示你手动粘贴 openId 或包含 openId 的 URL。

### 3. 运行效果

- **普通/GPS 签到**：全自动完成，无需任何操作
- **二维码签到**：终端会打印二维码，用微信扫码即可
- **openId 过期**：程序发送桌面通知提醒，自动从剪贴板等待新的 openId

## 致谢

基于 [@ManiaciaChao](https://github.com/ManiaciaChao) 的 [yatm](https://github.com/ManiaciaChao/yatm) 项目。

## 许可证

MIT
