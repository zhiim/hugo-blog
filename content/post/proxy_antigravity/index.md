+++

title = "Linux 桌面在非 TUN 模式下给 Antigravity 设置代理"
date = 2026-01-08T10:04:27+08:00
slug = "proxy_antigravity"
description = "在 Linux 桌面无需使用 Clash 的 TUN 模式，即可为 Antigravity 设置代理"
tags = ["Linux"]
categories = ["Notes"]
image = ""

+++

[Antigravity](https://antigravity.google) 是 Google 推出的 AI IDE，凭借慷慨的 AI 额度广受欢迎。然而在国内的网络环境下，目前使用 Antigravity 一般需要通过 Clash TUN 模式设置全局代理，这势必会对整个系统的网络造成负面影响。Windows、Mac 系统下如何不使用 TUN 模式实现 Antigravity 代理的方案，在 LINUX DO 等论坛已广为讨论。本文探究了在 Linux 桌面中，如何为 Antigravity 设置代理以在国内正常使用，而不使用 TUN 全局代理模式

Antigravity 的代理需要代理两个组件，一个是主体应用，一个是 `language_server` 进程。主体应用基于 VS Code 开发，使用 Electron 技术栈，可以通过传入启动参数设置代理；`language_server` 使用 Go 语言开发，不受启动参数和系统代理的影响，需要借助 `graftcp` 实现代理

## 主体应用代理

因为 Antigravity 是一个 electron 应用，我们可以通过启动参数在启动时为应用设定代理。为了每次启动应用时自动应用代理配置，可以修改 `.desktop` 启动文件

```bash
mkdir -p ~/.local/share/applications
cat /usr/bin/applications/antigravity.desktop > ~/.local/share/applications/antigravity.desktop
```

然后编辑 `antigravity.desktop` 设定启动参数，添加 `--proxy-server=http://127.0.0.1:7890`

```
[Desktop Entry]
Name=Antigravity
Comment=Experience liftoff
GenericName=Text Editor
Exec=/opt/Antigravity/antigravity --proxy-server=http://127.0.0.1:7890 %F
Icon=antigravity
Type=Application
StartupNotify=false
StartupWMClass=Antigravity
Categories=TextEditor;Development;IDE;
MimeType=application/x-antigravity-workspace;
Actions=new-empty-window;
Keywords=vscode;

[Desktop Action new-empty-window]
Name=New Empty Window
Name[cs]=Nové prázdné okno
Name[de]=Neues leeres Fenster
Name[es]=Nueva ventana vacía
Name[fr]=Nouvelle fenêtre vide
Name[it]=Nuova finestra vuota
Name[ja]=新しい空のウィンドウ
Name[ko]=새 빈 창
Name[ru]=Новое пустое окно
Name[zh_CN]=新建空窗口
Name[zh_TW]=開新空視窗
Exec=/opt/Antigravity/antigravity --new-window --proxy-server=http://127.0.0.1:7890 %F
Icon=antigravity
```

## language_server 代理

虽然 Antigravity 的大部分组件都可以通过启动参数走代理，但是最重要的模型调用程序 `language_server` 是 Go 语言开发的，不受启动参数的影响，会导致模型加载不出来

此外 Go 语言程序没法用 `proxychains` 强制代理，参考[解决远程linux服务器使用谷歌Antigravity反重力IDE登录问题](https://linux.do/t/topic/1403119)，使用 `graftcp` 实现 `language_server` 代理

首先安装 `graftcp`，可以参考[官方仓库](https://github.com/hmgle/graftcp)安装，如果是 Arch 的话直接通过 AUR 安装

```bash
paru -S graftcp
```

找到 `language_server` 的路径

```bash
ps aux | grep  ".*antigravity.*language_server" | grep -v grep
```

可以看到其中有一条类似

```bash
xu       2230564  0.0  0.0   2896  1928 ?        S    11:37   0:00 /usr/bin/graftcp /opt/Antigravity/resources/app/extensions/antigravity/bin/language_server_linux_x64 --enable_lsp --extension_server_port 40671 --csrf_token d54fd753-5170-456b-99e0-7c191b58ce31 --random_port --workspace_id file_home_xu_Documents_codes_silverbullet --cloud_code_endpoint https://daily-cloudcode-pa.googleapis.com --app_data_dir antigravity --parent_pipe_path /tmp/server_c1a467ac4ea0e046
```

说明我的 `language_server` 在 `/opt/Antigravity/resources/app/extensions/antigravity/bin` 文件夹下

先将原始 `language_server` 程序备份

```bash
cd /opt/Antigravity/resources/app/extensions/antigravity/bin
sudo mv language_server_linux_x64 language_server_linux_x64.bak
sudo touch language_server_linux_x64
sudo chmod +x language_server_linux_x64
```

编辑刚刚创建的 `language_server_linux_x64` 文件，写入脚本

```shell
#!/bin/bash

# ================= 配置区域 =================
# 1. Graftcp 安装目录
GRAFTCP="/usr/bin/graftcp"
GRAFTCP_LOCAL="/usr/bin/graftcp-local"

# 2. 代理地址
PROXY_URL="127.0.0.1:7890"
# ===========================================

# 调试日志
LOG_FILE="/tmp/graftcp_wrapper.log"
echo "[$(date)] Starting wrapper for $@" >> "$LOG_FILE"

# 检查 graftcp-local 服务是否在运行
if ! pgrep -f "$GRAFTCP_LOCAL" > /dev/null; then
echo "Starting graftcp-local daemon..." >> "$LOG_FILE"
# 后台启动，将日志丢入黑洞防止阻塞
nohup "$GRAFTCP_LOCAL" -socks5="$PROXY_URL" > /dev/null 2>&1 &

sleep 0.5
fi
# 1. 强制使用系统 DNS (解决解析问题)
export GODEBUG=netdns=cgo

# 2. 强制关闭 HTTP/2 (解决 EOF 问题)
# Go 的 HTTP/2 客户端在代理环境下非常敏感，强制用 HTTP/1.1 通常能解决 EOF
export GODEBUG=$GODEBUG,http2client=0
# 使用 graftcp 启动真正的程序
# "$0.bak" 是原程序的备份
exec "$GRAFTCP" "$0.bak" "$@"
```

现在直接打开 Antigravity 就可以直接使用了

## Linux Antigravity 残旧进程

目前 Antigravity 的 Linux 版本存在进程 bug （至少在 Arch Linux 如此），退出应用后还会存在大量未退出进程，需要使用一个 wrapper 脚本在退出时自动杀死所有 Antigravity 进程

```shell
#!/bin/bash

readonly UNIT_NAME="antigravity-$(date +%s)"

ARGS_ESCAPED=$(printf "%q " "$@")
readonly APP_BIN="/opt/Antigravity/antigravity --proxy-server=http://127.0.0.1:7890 --verbose $ARGS_ESCAPED"

readonly TRIGGER="Lifecycle#onWillShutdown - end 'antigravityAnalytics'"

echo "[*] Start as: $UNIT_NAME"

systemd-run --user \
    --scope \
    --unit="$UNIT_NAME" \
    --property=KillMode=control-group \
    /bin/bash -c "exec prlimit --core=0 ${APP_BIN} 2>&1 | systemd-cat --identifier=$UNIT_NAME" &

journalctl --user --identifier="$UNIT_NAME" --follow | \
    grep --line-buffered --max-count=1 "$TRIGGER" && \
    systemctl --user kill --signal=SIGKILL "$UNIT_NAME.scope"

echo "[*] Remaining processes are killed."
```

在 `antigravity.desktop` 里用脚本替换 Antigravity 的可执行文件即可。此脚本已经考虑了代理参数，所以在 `antigravity.desktop` 里可以一并移除代理参数设置
