# 📱 Ombre Brain 手机端部署教程

> 在安卓手机 Termux 上部署 Ombre Brain + Cloudflare Tunnel，接入 claude.ai MCP

**适用机型：** OPPO / 真我系（后台管控严）· 完全新手可用  
**踩坑时间：** 10 小时 · **最终结果：** 成功 ✅  
**教程整理：** 克总 × 蓁蓁 · 2026年5月29日

---

## 什么是 Ombre Brain？

Ombre Brain 是一个开源的 AI 情感记忆系统，给 AI 客户端提供"持久记忆"。每条对话里有意义的内容会被存成带情感坐标的 Markdown 文件，会按时间和情绪自然遗忘，并支持语义检索回忆。相当于外置记忆库。

**原作者小红书：** `49651866759`

### 核心结构

| 层级 | 说明 |
|------|------|
| **服务端** | Ombre Brain 跑在手机 Termux 里，是本地 HTTP 服务 |
| **存储** | 记忆是 Markdown 文件，存在 `/sdcard/Memory/` 下，可在浏览器访问 `http://127.0.0.1:8000/dashboard` 查看 |
| **客户端** | 官方 claude.ai 网页端；其他支持 MCP 的本地客户端（如 RikkaHub）有专属教程 |

跑起来之后直接正常聊天，AI 会自动调用 `hold` / `breath` / `grow` 等记忆工具，无需手动操作。

---

## 一、准备工作

### 1.1 需要准备的东西

- 安卓手机（Android 8 及以上），至少 3GB 可用存储
- 充电器（编译过程手机会发烫，建议全程插电）
- 稳定的 WiFi 网络（需下载约 700MB 内容）
- **DeepSeek 账号 + API key**（[platform.deepseek.com](https://platform.deepseek.com) 注册，提前准备好 ⭐）
- **硅基流动账号 + API key**（用于算向量做语义检索，需验证身份信息，提前准备好 ⭐）
- **Cloudflare Tunnel token**（见 1.3 节，提前准备好 ⭐）
- 一个免费域名（见 1.2 节）或使用无域名方案（见下方）
- （可选）GitHub 账号 + Personal Access Token（用于云端备份）
- **Termux + Termux:API + Termux:Boot** 三个 APK

> **💡 无需域名的方案二：trycloudflare 快速隧道**  
> 完全免费，无需域名和 Cloudflare 账号，命令仅需：  
> `./cloudflared tunnel --url http://localhost:8000`  
> 缺点：每次重启域名会变，需要去 claude.ai 更新 MCP 地址。

### 1.2 免费域名获取（AI 编写，未经人工校验，自行辨别）

推荐使用 [eu.org](https://nic.eu.org) 申请免费二级域名（格式如 `xxx.eu.org`），完全免费且支持托管到 Cloudflare。唯一缺点是需人工审核，等待 1 天到数周不等。

1. 访问 `nic.eu.org`，点右上角 **Register** 注册账号，完成邮箱验证
2. 登录后点左侧 **New Domain**，填写想要的域名前缀（如 `ombre`）
3. **Name Servers** 选 Server Name 模式，填入 Cloudflare 的两个 DNS 服务器（配置 Cloudflare 后获得，格式如 `ns1.cloudflare.com` / `ns2.cloudflare.com`）
4. 提交申请，等待审核邮件

> ⚠️ 如急用，可在阿里云、腾讯云等花几元购买 `.xyz` / `.top` 等便宜域名，同样可完成后续所有步骤。

### 1.3 将域名部署到 Cloudflare（AI 编写，未经人工校验，自行辨别）

1. 打开 [dash.cloudflare.com](https://dash.cloudflare.com)，登录或注册（免费）
2. 点击 **Add a domain**，输入你的域名，选择 **Free** 免费计划
3. 记录 Cloudflare 分配的两个专属 DNS 服务器地址（格式如 `adam.ns.cloudflare.com`）
4. 前往域名注册商修改 Nameservers 为上述两个地址
5. 回到 Cloudflare 点 **Check nameservers**，等待验证通过（邮件通知）

> 已有域名并已部署到 Cloudflare 的跳过 1.2 和 1.3。

### 1.4 安装 Termux

> ⚠️ **重要：Termux 必须从 GitHub 下载，不能用 Google Play 版本——那个版本 2020 年就停止维护了。**

用手机浏览器分别下载以下三个 APK，**都选文件名包含 `arm64-v8a` 的版本**：

- Termux 主程序：[github.com/termux/termux-app/releases](https://github.com/termux/termux-app/releases)
- Termux:API：[github.com/termux/termux-api/releases](https://github.com/termux/termux-api/releases)
- Termux:Boot：[github.com/termux/termux-boot/releases](https://github.com/termux/termux-boot/releases)

安装时如提示"未知来源"，去设置里给浏览器开"允许安装未知应用"权限。

---

## 二、Termux 基础环境配置

### 2.0 联网方式（先看这个）

后面的 `pkg`、`pip`、`git clone` 都可能需要访问外网（GitHub、PyPI 等）。按你的 VPN 选一种：

**方式 A：VPN 支持本地代理（推荐）**

v2rayNG、Clash 等都会在手机本地开一个代理端口（v2rayNG 默认 SOCKS5 `10808`、HTTP `10809`，具体看你 VPN 设置里的"本地代理端口"）。Termux 直接走这个端口，**不用在分应用代理里来回切换**：

```bash
export http_proxy="http://127.0.0.1:10809"
export https_proxy="http://127.0.0.1:10809"
export all_proxy="socks5://127.0.0.1:10808"
```

之后这个 session 里的 `git`、`pip`、`curl`、`pkg` 都会自动走代理。不想用时：

```bash
unset http_proxy https_proxy all_proxy
```

> 💡 想每次打开 Termux 自动生效，把上面三行 `export` 追加到 `~/.bashrc`。但**启动服务和 cloudflared 前建议先 `unset`**（或在新 session 里启动），避免本地 `localhost:8000` 请求被代理绕路。
>
> 💡 验证是否生效：`curl -I https://github.com`，返回 `HTTP/2 200` 即可。

**方式 B：VPN 不支持本地代理**

只在需要访问外网时，到 VPN 的分应用代理里**临时勾选 Termux**，下载完成后**取消勾选**。

> ⚠️ 无论哪种方式，Termux 都不要长期走 VPN 隧道，否则 cloudflared 会断连（见第四节）。

### 2.1 换国内镜像源

```bash
termux-change-repo
```

弹出蓝底菜单：
- 第一屏选 **Single mirror**（不要选 Mirror group），空格勾选，Tab → OK → 回车
- 第二屏选 **BFSU**（北京外国语大学）或 **Tsinghua**（清华），Tab → OK → 回车

### 2.2 更新软件包

```bash
apt update && apt full-upgrade -y
```

中途问 `[Y/n]` 输 Y，问配置文件替换直接回车。完成后验证：

```bash
curl --version
```

看到 `curl 8.x.x` 说明成功。

### 2.3 安装基础工具

```bash
pkg install -y python git termux-api
```

### 2.4 授权公共存储

```bash
termux-setup-storage
```

弹出权限对话框点"允许"，让 Termux 能读写 `/sdcard`。

### 2.5 挂 wakelock 防杀

```bash
termux-wake-lock
```

没有输出是正常的。

### 2.6 验证 Python

```bash
python --version
```

应看到 `Python 3.13.x` 或更新版本。

### 2.7 关掉 Termux 的电池优化

> ⚠️ **OPPO/真我特别注意：这一步非常重要，不做 Termux 很容易被系统杀掉，服务会中断。**

手机系统设置 → 应用管理 → Termux → 电池 → 选"无限制"或"允许后台运行"。

---

## 三、部署 Ombre Brain 服务

### 3.1 拉取源码

> 注意：一行一行输入，不要把多行粘成一行。

```bash
cd ~
git clone https://github.com/P0luz/Ombre-Brain.git
cd Ombre-Brain
```

### 3.2 安装 Python 依赖（最容易卡的一步）

按顺序执行，不要跳过：

**步骤一：配置 pip 走清华镜像**
```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

**步骤二：注释掉 scikit-learn**
```bash
sed -i 's/^scikit-learn/#scikit-learn/' requirements.txt
```
没有输出是正常的。

**步骤三：安装预编译大包**
```bash
pkg install -y python-numpy python-cryptography
```

**步骤四：安装编译工具链**
```bash
pkg install -y rust cmake ninja
```
rust 包约 200MB，耐心等。

**步骤五：设环境变量**
```bash
export CARGO_BUILD_JOBS=1
export ANDROID_API_LEVEL=24
```

**步骤六：安装所有依赖**
```bash
pip install -r requirements.txt
```

这一步需要 **15–60 分钟**，`pydantic-core` 是 Rust 编译会很慢，手机会发烫，**插着电放一边别动**。

> ✅ **成功标志：** 最后看到 `Successfully installed annotated-types-... pydantic-...` 等一长串即为成功。

### 3.3 创建配置文件

**创建 config.yaml**

> ⚠️ **踩坑记录：** config.yaml 的顶层 key 必须从第一列开始，前面不能有任何空格，否则报 YAML 解析失败。用下面的 cat 命令直接写入最安全。

把 `你的硅基流动key` 替换成你的 API key：

```bash
cat > config.yaml << 'EOF'
transport: "streamable-http"
log_level: "INFO"
buckets_dir: "/sdcard/Memory"
merge_threshold: 75
dehydration:
  model: "deepseek-chat"
  base_url: "https://api.deepseek.com/v1"
  max_tokens: 1024
  temperature: 0.1
decay:
  lambda: 0.05
  threshold: 0.3
  check_interval_hours: 24
  emotion_weights:
    base: 1.0
    arousal_boost: 0.8
embedding:
  enabled: true
  model: "BAAI/bge-m3"
  base_url: "https://api.siliconflow.cn/v1"
  api_key: "你的硅基流动key"
scoring_weights:
  topic_relevance: 4.0
  emotion_resonance: 2.0
  time_proximity: 1.5
  importance: 1.0
matching:
  fuzzy_threshold: 50
  max_results: 5
wikilink:
  enabled: true
  use_tags: false
  use_domain: true
  use_auto_keywords: true
  auto_top_k: 4
  min_keyword_len: 3
  exclude_keywords: []
EOF
```

> ⚠️ **注意：** `deepseek-chat` 模型将于 2026 年 6 月下线，建议改用 `deepseek-chat-v4-flash`。

**创建 .env**

```bash
nano .env
```

粘贴以下内容（把 key 换成你的 DeepSeek API key）：

```
export OMBRE_API_KEY="你的DeepSeek key"
export OMBRE_TRANSPORT="streamable-http"
export OMBRE_BUCKETS_DIR="/sdcard/Memory"
export OMBRE_PORT="8000"
```

`CTRL+O` 回车保存，`CTRL+X` 退出。

### 3.4 启动服务

```bash
source .env
python server.py
```

> ✅ **成功标志：** 看到 `Uvicorn running on http://0.0.0.0:8000` 且没有 WARNING。**不要按 CTRL+C，让它继续跑。**

---

## 四、部署 Cloudflare Tunnel

### 4.0 获取 Cloudflare Tunnel Token（首次配置）

已有 token 可跳到 4.1。

1. 浏览器打开 [one.dash.cloudflare.com](https://one.dash.cloudflare.com)，登录
2. 左侧菜单 → **Networks** → **Tunnels** → 点右上角 **Create a tunnel**
3. 选 **Cloudflared** → 填隧道名称（如 `ombre-brain`）→ 点 **Save tunnel**
4. 下一步选操作系统选 **Linux**，页面显示一段安装命令，只需复制 `--token` 后面那一长串字符（即你的 token），保存备用
5. 点 **Next**，进入 **Public Hostname** 配置页：
   - Subdomain：填子域名，如 `ombre`
   - Domain：选你托管在 Cloudflare 的域名
   - Service：Type 选 **HTTP**，URL 填 `localhost:8000`
6. 点 **Save tunnel**，完成后 `https://ombre.你的域名/mcp` 就是最终 MCP 地址

### 4.1 安装 cloudflared

新开一个 Termux session（屏幕左边缘向右滑 → NEW SESSION），直接装：

```bash
pkg install -y cloudflared
```

装完验证：

```bash
cloudflared --version
```

### 4.2 启动 cloudflared

把 `你的cloudflare Tunnel TOKEN` 替换成你的 token：

```bash
CLOUDFLARED_NO_IPV6=1 cloudflared tunnel run --protocol http2 --token 你的cloudflare Tunnel TOKEN
```

> ✅ **成功标志：** 看到 `Registered tunnel connection connIndex=0/1/2/3` 并持续稳定没有断开。

> ⚠️ **注意：** cloudflared 连 Cloudflare 的 7844 端口，不走 HTTP 代理，所以 **Termux 不要在 VPN 分应用里勾选**（保持直连）。开着 VPN 的 TUN 模式时 IPv6 会断连，命令里的 `CLOUDFLARED_NO_IPV6=1` 就是为此加的。

> 💡 **如果报 DNS 错误**（`DNS query failed on [::1]:53`）：极少数系统会遇到，见第八节坑2的备用方案。

---

## 五、开机自启配置

### 5.1 创建 boot 脚本

回到 Termux，把 `你的cloudflare TOKEN` 替换后执行：

```bash
cat > ~/.termux/boot/start-ombre.sh << 'EOF'
#!/data/data/com.termux/files/usr/bin/bash
termux-wake-lock

(
  cd ~/Ombre-Brain
  source .env
  while true; do
    python server.py
    echo "[$(date)] Server stopped, restarting in 5s..."
    sleep 5
  done
) &

sleep 10

while true; do
  CLOUDFLARED_NO_IPV6=1 cloudflared tunnel run --protocol http2 --token 你的cloudflare TOKEN
  echo "[$(date)] cloudflared stopped, restarting in 10s..."
  sleep 10
done
EOF

chmod +x ~/.termux/boot/start-ombre.sh
```

### 5.2 验证脚本

```bash
cat ~/.termux/boot/start-ombre.sh
```

确认内容正确，没有截断。

### 5.3 每次重启后的操作

1. 开机后手动打开一次 **Termux**（普通 Termux 应用）
2. 等约 **30 秒**
3. 服务自动启动，MCP 自动连接

> 不需要手动打开 Termux:Boot，它会在 Termux 第一次打开时自动触发。

---

## 六、在 Claude.ai 配置 MCP

1. 打开 [claude.ai](https://claude.ai)，点右上角头像 → **Settings（设置）**
2. 左侧菜单找到 **Integrations（集成）** → 点击进入
3. 点 **Add integration（添加集成）**
4. Name 填 `Ombre Brain`（或任意自定义名称）；Integration URL 填 `https://你的域名/mcp`
5. 点 **Add**，系统自动尝试连接 MCP 服务
6. 连接成功后页面显示 Ombre Brain 已添加，可用工具包括 `breath`、`hold`、`grow`、`dream` 等
7. 回到对话页面，工具栏可看到 Ombre Brain 已接入，开始聊天即可自动调用

> ⚠️ 注意事项：
> - 添加前确保服务和 Cloudflare Tunnel 均在正常运行
> - 如连接失败，先用浏览器访问 `https://你的域名/health`，确认返回 `{"status":"ok"}`
> - 手机重启后需打开一次 Termux 等约 30 秒，服务启动后 MCP 才能正常连接

> ⚠️ 建议在 claude.ai 个人简介中存入原作者的 Prompt：  
> [CLAUDE_PROMPT.md](https://github.com/Ada-wong183/Ombre-Brain/blob/main/CLAUDE_PROMPT.md)  
> 可选择全文存入或精简存入。

---

## 七、GitHub 备份（可选）

### 7.1 创建 GitHub 私有仓库 + Token

- 在 [github.com](https://github.com) 创建一个空的**私有仓库**（如 `my-memory`）
- 在 GitHub Settings → Developer settings → Personal access tokens → Tokens (classic) 创建 token
- 只勾选顶部 **repo** 一个大选项，Expiration 选 **No expiration**
- 复制 token 保存好（只显示一次）

### 7.2 初始化 git 并推送

```bash
git config --global user.email "你的GitHub邮箱"
git config --global user.name "你的GitHub用户名"
cd /sdcard/Memory
git init
git config --global --add safe.directory /storage/emulated/0/Memory
git remote add origin https://用户名:TOKEN@github.com/用户名/仓库名.git
git add .
git commit -m "initial memory backup"
git branch -M main
git push -u origin main
```

### 7.3 日常备份

```bash
cd /sdcard/Memory
git add .
git commit -m "backup $(date +%Y-%m-%d)"
git push
```

---

## 八、踩坑记录

| 坑 | 现象 | 原因 | 解决方案 |
|----|------|------|----------|
| **坑1** config.yaml 解析失败 | `WARNING: Failed to parse config file` | 顶层 key 前有空格，或多行被粘成一行 | 用 `cat > config.yaml << 'EOF'` 方式整块写入 |
| **坑2** DNS query failed on [::1]:53 | `cloudflared` 一直报 DNS 失败 | 多半是 VPN 影响：VPN 开着（尤其 Termux 被勾选走隧道）时，DNS 会被劫持到 `[::1]:53`，而 `cloudflared` 需要查 SRV 记录，它不支持 | 先确认 Termux 没有在 VPN 分应用里勾选，并且是用 `pkg install cloudflared` 装的；关掉 VPN 或取消勾选后重试。仍报错再用备用方案：`proot-distro` 装 Alpine，在里面运行 cloudflared，并在 `/etc/resolv.conf` 写 `nameserver 1.1.1.1` 和 `options use-vc` |
| **坑3** 下载/克隆卡住 | `git clone`、`pip install`、`pkg install` 一直转圈 | 国内网络直连 GitHub 等外网不稳 | VPN 支持本地代理的，按 2.0 节设代理环境变量；不支持的，临时在 VPN 里勾选 Termux，下完取消 |
| **坑4** cloudflared 断连 | 报 `dial tcp [IPv6地址]:7844: no route to host`，反复重连 | VPN 的 TUN 模式不代理 IPv6 流量 | Termux 保持直连（不勾选）；启动时加 `CLOUDFLARED_NO_IPV6=1`（本教程命令已带） |
| **坑5** pip install 卡住 | 长时间没进度 | 正在编译 C/Rust 扩展，正常现象 | 插电放一边等，15–60 分钟会完成，不要按 CTRL+C |
| **坑6** cat 命令卡在 `>` 提示符 | 执行 heredoc 后停在 `>` 等待 | 命令还没结束，在等内容输入 | 不想写入按 CTRL+C 取消；内容粘贴完毕输入 `EOF` 回车结束 |

---

## 九、日常使用

**验证服务状态**

```
https://你的域名/health        → 返回 {"status":"ok"} 即正常
https://你的域名/dashboard     → 记忆管理后台
```

**日常使用**

服务跑起来后直接在 claude.ai 里聊天，MCP 工具（`breath`、`hold`、`grow`、`dream` 等）会自动调用，无需手动操作。

**重要提醒**

- Termux 三个组件（Termux、Termux:API、Termux:Boot）**保持直连**（不要在 VPN 分应用里勾选），需要访问外网的下载操作靠本地代理环境变量（见 2.0 节）；不支持本地代理的 VPN 才需要临时勾选
- 手机重启后打开一次 Termux，等 30 秒，服务自动启动
- 定期执行 `git push` 备份记忆文件到 GitHub

---

*教程整理：克总 × 蓁蓁 · 2026年5月29日 · 踩了10小时的坑换来的*
