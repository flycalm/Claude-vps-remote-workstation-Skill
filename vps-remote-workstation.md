---
name: vps-remote-workstation
description: 将低配 Linux VPS（如 2C4G Ubuntu 24.04）改造为海外远程桌面工作站的完整指南與 Agent Skill —— 涵盖 XFCE4+xrdp 轻量级桌面、Mihomo TUN 全局接管与 SSH/RDP 防失联策略路由、机场前置与静态住宅 ISP 链式代理（dialer-proxy）、按域名/进程精准分流，以及在无物理显卡云服务器上通过原生 Firefox ESR 绕过 Cloudflare Turnstile / Claude 等严苛登录风控的实战方案。
---

# 低配 VPS 海外远程桌面工作站搭建与防风控指南

本指南总结自 2C4G Ubuntu 24.04 无显卡云服务器的实战经验：通过 **XFCE4 + xrdp** 提供极低内存占用的远程桌面，使用 **Mihomo (Clash.Meta) TUN 模式**接管全局流量，配合 **Linux 策略路由锁**防止 SSH/RDP 断连失联，并通过**链式静态 ISP 代理 + 原生 Firefox ESR 独立 Profile**，解决无 GPU 虚拟机在登录 Claude 等严苛 SaaS 平台时频繁遭遇 `Couldn't complete sign-in. You can try signing in from a different browser.` 的风控拦截问题。

---

## 0. 防灾难安全铁律（必读）

1. **修改任何配置前先备份**：
   `cp -a <file> <file>.bak.$(date +%Y%m%d-%H%M%S)`
2. **未配置防失联路由前，绝对不要启动 TUN 模式**：
   启动前必须用 `ip route get <你的客户端IP> sport 22` 验证回程路由，确保显示 `dev eth0`（主网卡直连），绝不能是 `MihomoTUN`。
3. **保留云厂商后台 VNC 救援通道**：
   在确认 SSH 和远程桌面稳定之前，保持云服务商网页端的 VNC 控制台可用。
4. **重启服务前必须校验配置语法**：
   每次修改后执行 `mihomo -t -d /etc/mihomo`，确认无语法错误后再 `systemctl restart mihomo`。

---

## 1. 系统底层基线与环境自洽

海外 SaaS 平台的风控系统会交叉比对 **出口 IP 归属地**、**系统语言环境（Locale）** 与 **浏览器 JS 时区（`Intl.DateTimeFormat`）**。系统时区必须与**最终落地出口 IP 所在的物理州/城市**严格对齐（例如落地在美东弗吉尼亚/纽约则设为 `America/New_York`，落地在美西洛杉矶则设为 `America/Los_Angeles`）。

```bash
# 1. 彻底禁用 IPv6（防止服务器原生 IPv6 绕过代理泄漏真实机房位置）
cat << 'EOF' | sudo tee /etc/sysctl.d/99-disable-ipv6.conf
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
sudo sysctl --system

# 2. 锁定纯英文语言环境，并将时区对齐到落地 IP 所在时区
sudo locale-gen en_US.UTF-8
sudo update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 LANGUAGE=en_US:en
sudo timedatectl set-timezone America/New_York   # 请根据实际落地 IP 所在美东/美西修改
sudo timedatectl set-ntp true
sudo hostnamectl set-hostname workstation-node

# 3. 开启 4GB Swap 虚拟内存（2C4G 机器运行桌面+浏览器防 OOM 崩溃必配）
if [ $(swapon --show | wc -l) -eq 0 ]; then
    sudo fallocate -l 4G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.d/99-swappiness.conf
    sudo sysctl --system
fi
```

---

## 2. Mihomo TUN 模式与 SSH/RDP 防失联锁（核心步骤）

在远程服务器上开启 `tun.auto-route: true` 会劫持默认路由表。如果不提前锁定 SSH（默认 22）和 RDP（3389）的源端口回程路由，服务器发回给你本地电脑的响应包会被塞进代理隧道，导致**瞬间断连失联**。

### 2.1 创建防失联策略路由脚本 `/usr/local/bin/protect-ssh.sh`

```bash
cat << 'EOF' | sudo tee /usr/local/bin/protect-ssh.sh
#!/bin/bash
SSH_PORT=$(ss -tlnp | grep sshd | awk '{print $4}' \vert{} awk -F: '{print$NF}' | head -n 1)
SSH_PORT=${SSH_PORT:-22}

# 优先级 100/101 高于 Mihomo TUN 的路由表（约 9000），强制管理端口回包走 main 表直连
ip rule del sport $SSH_PORT table main pref 100 2>/dev/null || true
ip rule add sport $SSH_PORT table main pref 100

ip rule del sport 3389 table main pref 101 2>/dev/null || true
ip rule add sport 3389 table main pref 101
EOF
sudo chmod +x /usr/local/bin/protect-ssh.sh
sudo /usr/local/bin/protect-ssh.sh
```

### 2.2 配置 Systemd 服务并绑定前置保护钩子

```ini
# /etc/systemd/system/mihomo.service
[Unit]
Description=Mihomo Daemon
After=network.target NetworkManager.service systemd-networkd.service

[Service]
Type=simple
User=root
ExecStartPre=/usr/local/bin/protect-ssh.sh
ExecStart=/usr/local/bin/mihomo -d /etc/mihomo
Restart=on-failure
RestartSec=5
LimitNOFILE=1048576

[Install]
WantedBy=multi-user.target
```

---

## 3. 出口网络架构：机场前置加速 + 静态 ISP 链式分流

### 为什么普通机场/VPS 节点无法通过严苛登录验证？
绝大多数机场订阅和云服务器自带的 IP 属于数据中心机房段（如 `ColoCrossing`、`HostPapa`、`OVH`、`G-Core` 等），在 IP 数据库中标记为 `hosting`。Cloudflare Turnstile 和 Stytch 等反欺诈引擎会对机房 ASN 施加最高等级拦截。

**高性价比解决方案（链式代理 `dialer-proxy`）**：
* **日常大流量**（系统更新、普通网页）：继续走高速且廉价的**机场订阅美国节点**。
* **敏感鉴权与目标应用流量**（如 Claude）：购买一个单月几元至几美元的**美国静态住宅/本土固网 ISP 代理（SOCKS5）**，利用 Mihomo 的 `dialer-proxy` 让机场节点作为“前置跳板”去拨号连接该静态 ISP 代理。既隐藏了 VPS 本机 IP，又保证了最终露面的是干净的美国本土固网 ISP（如 Verizon、AT&T、Frontier、Comcast），同时避免跑满静态住宅代理的限速带宽。

### `/etc/mihomo/config.yaml` 完整配置模板

```yaml
mixed-port: 7890
allow-lan: false
mode: rule
log-level: info
ipv6: false
external-controller: 127.0.0.1:9090
find-process-mode: strict

listeners:
  # 绑定到纯净静态 ISP 出口的本地专属监听端口（供特定浏览器 Profile 直接调用）
  - name: isp-dedicated-out
    type: mixed
    listen: 127.0.0.1
    port: 7891
    proxy: "US-ISP-Clean"

profile:
  store-selected: true
  store-fake-ip: false

sniffer:
  enable: true
  parse-pure-ip: true
  sniff:
    HTTP: {ports: [80, 8080-8880]}
    TLS: {ports: [443, 8443]}

tun:
  enable: true
  stack: mixed
  device: MihomoTUN
  auto-route: true
  auto-redirect: true
  auto-detect-interface: true
  strict-route: true
  dns-hijack:
    - any:53
    - tcp://any:53
  route-exclude-address:
    - 10.0.0.0/8
    - 172.16.0.0/12
    - 192.168.0.0/16

dns:
  enable: true
  listen: 0.0.0.0:1053
  ipv6: false
  enhanced-mode: redir-host
  default-nameserver: [1.1.1.1, 8.8.8.8]
  nameserver:
    - [https://1.1.1.1/dns-query#GLOBAL-US](https://1.1.1.1/dns-query#GLOBAL-US)
    - [https://8.8.8.8/dns-query#GLOBAL-US](https://8.8.8.8/dns-query#GLOBAL-US)

proxy-providers:
  us-airport:
    type: http
    url: "[https://your-subscription-url.example.com/sub](https://your-subscription-url.example.com/sub)"
    interval: 86400
    path: ./sub.yaml
    # 正则过滤：仅保留订阅中的美国节点
    filter: "(?i)(US|USA|United States|America|美国|美西|美东|洛杉矶|圣何塞|西雅图|纽约|芝加哥|达拉斯|硅谷)"
    exclude-filter: "(?i)(HK|香港|TW|台湾|JP|日本|SG|新加坡|KR|韩国|[v6])"
    health-check:
      enable: true
      url: [https://www.gstatic.com/generate_204](https://www.gstatic.com/generate_204)
      interval: 300

proxies:
  # 静态住宅/ISP SOCKS5 节点，通过 US-TRANSIT-DIAL 代理组进行前置链式拨号
  - name: "US-ISP-Clean"
    type: socks5
    server: <YOUR_ISP_PROXY_HOST>
    port: <YOUR_ISP_PROXY_PORT>
    username: "<YOUR_ISP_PROXY_USER>"
    password: "<YOUR_ISP_PROXY_PASS>"
    udp: false
    dialer-proxy: "US-TRANSIT-DIAL"

proxy-groups:
  # 包装一层 select 组供 dialer-proxy 调用，避免订阅节点名称带国旗 Emoji 或倍率后缀导致解析失败
  - name: "US-TRANSIT-DIAL"
    type: select
    use: [us-airport]

  - name: "GLOBAL-US"
    type: select
    use: [us-airport]

rules:
  # 1. 【极其关键】第一条必须拒绝 QUIC (UDP 443)！
  # 因为大多数 SOCKS5 静态住宅代理不支持 UDP (udp: false)，若域名规则在前，QUIC 包会掉入黑洞超时；
  # 放在第一条直接 REJECT，可让浏览器瞬间回退到稳定的 TCP。
  - AND,((NETWORK,udp),(DST-PORT,443)),REJECT

  # 2. 【同源鉴权分流】将目标站与所有内嵌人机验证/鉴权/风控域名统一路由至同一个静态 ISP 出口
  # 若主站走 ISP 出口，而内嵌的 challenges.cloudflare.com 走了机房出口，会导致跨 ASN 令牌校验直接失败！
  - DOMAIN-SUFFIX,claude.ai,US-ISP-Clean
  - DOMAIN-SUFFIX,anthropic.com,US-ISP-Clean
  - DOMAIN-SUFFIX,challenges.cloudflare.com,US-ISP-Clean
  - DOMAIN-SUFFIX,stytch.com,US-ISP-Clean
  - DOMAIN-SUFFIX,statsig.com,US-ISP-Clean
  - DOMAIN-SUFFIX,statsigapi.net,US-ISP-Clean
  - DOMAIN-SUFFIX,arkoselabs.com,US-ISP-Clean
  - DOMAIN-SUFFIX,funcaptcha.com,US-ISP-Clean
  - DOMAIN-SUFFIX,stripe.com,US-ISP-Clean
  - DOMAIN-SUFFIX,intercom.io,US-ISP-Clean
  - DOMAIN-SUFFIX,sentry.io,US-ISP-Clean
  - DOMAIN-KEYWORD,claude,US-ISP-Clean

  # 3. 进程级分流：强制 Firefox ESR 的所有流量（含在其中打开的收件邮箱）全部走纯净 ISP 出口
  - PROCESS-NAME,firefox-esr,US-ISP-Clean

  # 4. 其余常规流量分流
  - GEOIP,CN,DIRECT,no-resolve
  - GEOIP,PRIVATE,DIRECT,no-resolve
  - MATCH,GLOBAL-US
```

---

## 4. 轻量级远程桌面部署（XFCE4 + xrdp）

在 2C4G 无显卡 VPS 上切勿安装臃肿的 Ubuntu 默认 GNOME 桌面（空载占用超 1.5GB）。使用 **XFCE4 + xrdp** 空载仅占约 380MB 内存：

```bash
sudo DEBIAN_FRONTEND=noninteractive apt update
sudo DEBIAN_FRONTEND=noninteractive apt install -y xfce4 xfce4-goodies xorg dbus-x11 x11-xserver-utils xrdp
sudo adduser xrdp ssl-cert

# 创建普通桌面用户 dev（Chrome/Firefox 沙盒机制默认禁止以 root 身份直接运行）
sudo useradd -m -s /bin/bash -G sudo,ssl-cert dev
echo "dev:YourStrongPassword123!" | sudo chpasswd
echo "xfce4-session" | sudo -u dev tee /home/dev/.xsession
```

配置 `/etc/xrdp/startwm.sh`，确保每次远程桌面登录时自动继承纯英文 Locale 与匹配落地 IP 的时区：

```bash
sudo tee /etc/xrdp/startwm.sh << 'EOF'
#!/bin/sh
if [ -r /etc/profile ]; then
    . /etc/profile
fi
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
export LANGUAGE=en_US:en
export TZ=America/New_York
unset DBUS_SESSION_BUS_ADDRESS
unset XDG_RUNTIME_DIR
exec startxfce4
EOF
sudo chmod +x /etc/xrdp/startwm.sh
sudo systemctl enable --now xrdp
sudo systemctl restart xrdp
```

* **安全连接建议**：不要在公网直接裸露 3389 端口。在本地电脑终端通过 SSH 隧道转发连接：
  `ssh -L 13389:127.0.0.1:3389 root@<你的VPS_IP>`，然后使用 Windows 远程桌面（`mstsc`）或 macOS `Windows App` 连接 `127.0.0.1:13389`。

---

## 5. 突破无显卡 VPS 登录风控的核心实战（为什么弃 Chrome 用 Firefox ESR）

在无独立显卡（GPU-less）的 Linux 云服务器远程桌面中登录 Claude 等受 Cloudflare Turnstile / Stytch 保护的站点时，最常见的报错是：
> `Couldn't complete sign-in. You can try signing in from a different browser.`

### 5.1 三大踩坑死穴剖析

1. **死穴一：给 Linux 浏览器强行修改 `--user-agent` 伪装成 macOS/Windows（必死）**
   现代 Chrome 支持 `Sec-CH-UA-Platform`（Client Hints 客户端提示头），且网页 JS 会直接读取 `navigator.platform` 和底层渲染器。如果启动参数加了 `--user-agent` 声称自己是 macOS，但 Client Hints 和 JS 接口全部诚实返回 `Linux x86_64`，这种**自相矛盾的撕裂指纹**会被 Cloudflare 瞬间判定为伪造环境的爬虫。**务必保持原生 Linux 指纹的自洽！**
2. **死穴二：Chromium 在无物理显卡下的 `SwiftShader / llvmpipe` 高危标记**
   无显卡 VPS 上的 Chrome 使用软件渲染（WebGL 暴露为 `llvmpipe` 或 `Google SwiftShader`），配合 Chromium 的高熵 Client Hints，在点击登录触发静默算力挑战时极易被降权拒签。
3. **死穴三：跨设备看邮件 / 跨浏览器点 Magic Link 导致一次性令牌失效**
   如果在手机上点开登录验证邮件（或被手机邮箱 App 后台预加载扫描），再去电脑端填 6 位验证码或打开链接，会因为**发起端与验证端的 IP / TLS 指纹不一致**，被判定为异地会话劫持而直接拒签。

### 5.2 终极破局方案：原生 `firefox-esr` + 绑定专属 ISP 端口 + 同浏览器内收发邮件

改用 **非 Snap 版的原生 Firefox ESR（Gecko 内核）** 可以完美规避 Chromium 的 `Sec-CH-UA` 高熵审查，在无显卡 Linux 桌面上过盾率极高。

#### 第一步：通过 Mozilla 官方 PPA 安装原生 `firefox-esr`（避开 Ubuntu Snap 慢速容器）

```bash
sudo add-apt-repository -y ppa:mozillateam/ppa
cat << 'EOF' | sudo tee /etc/apt/preferences.d/mozillateam-ppa
Package: firefox* thunderbird*
Pin: release o=LP-PPA-mozillateam
Pin-Priority: 1001
EOF
sudo apt-get update && sudo apt-get install -y firefox-esr
```

#### 第二步：创建锁定 `7891` 纯净 ISP 端口的 Firefox 独立配置与桌面启动器

```bash
sudo -u dev mkdir -p /home/dev/.mozilla/firefox/work-clean
cat << 'EOF' | sudo -u dev tee /home/dev/.mozilla/firefox/work-clean/user.js
user_pref("network.proxy.type", 1);
user_pref("network.proxy.socks", "127.0.0.1");
user_pref("network.proxy.socks_port", 7891);
user_pref("network.proxy.socks_version", 5);
user_pref("network.proxy.socks_remote_dns", true);
user_pref("network.trr.mode", 5); // 禁用 DoH，防止 DNS 绕过本地代理造成解析泄漏
user_pref("intl.accept_languages", "en-US, en");
EOF

cat << 'EOF' | sudo -u dev tee /home/dev/Desktop/Firefox-Workstation.sh
#!/bin/bash
# 自动清理非正常关闭残留的锁文件，防止弹窗报错 Profile in use
rm -f /home/dev/.mozilla/firefox/work-clean/lock \
      /home/dev/.mozilla/firefox/work-clean/.parentlock \
      /home/dev/.mozilla/firefox/work-clean/.startup-incomplete
exec firefox-esr --no-remote --profile /home/dev/.mozilla/firefox/work-clean "[https://claude.ai/login](https://claude.ai/login)"
EOF
sudo chmod +x /home/dev/Desktop/Firefox-Workstation.sh
```

#### 第三步：100% 成功的「同实例闭环登录流」

1. 在远程桌面中双击运行 `Firefox-Workstation.sh`。
2. **核验出口与时区**：先访问 `https://claude.ai/cdn-cgi/trace`，确认 `ip=` 为你购买的美国静态 ISP 出口 IP，且 `colo=`（如美东 `IAD`、美西 `LAX`）与系统 `date` 命令显示的时区完全吻合。（*注：许多中转类 SOCKS5 代理的“入口连接 IP”与最终对外露面的“落地出口 IP”不同，务必以 `ipinfo.io` 测出的真实落地 IP 归属地为准来对齐系统时区*）。
3. **在同一个 Firefox 窗口中登录邮箱（如 Gmail）**：
   * 在当前 Firefox 窗口新建一个标签页，打开 `https://mail.google.com` 并登录你的接收邮箱。
   * 暂时关闭手机上的邮箱通知预览，防止手机端后台预加载消耗一次性链接。
4. **同浏览器内完成验证闭环**：
   * 在 Firefox 的第 1 个标签页（`claude.ai/login` 或 OAuth 授权页）输入邮箱发送验证邮件。
   * 切换到第 2 个标签页（Firefox 内的 Gmail），打开收到的邮件，**直接在当前 Firefox 中左键点击 “Sign in to Claude” 魔法链接**。
   * 由于**发起登录请求、加载 Turnstile 挑战框、打开 Gmail 邮件、以及点击消费 Magic Link** 全程发生在**同一个 Firefox 进程、同一个原生 Linux 指纹、同一个美区时区、以及同一个静态 ISP 出口 IP** 下，风控校验将一次性丝滑通过！

---

## 6. 验收与排障命令手册（Verification Runbook）

```bash
# 1. 检查 Mihomo 服务、TUN 网卡与 SSH/RDP 防失联锁状态
systemctl is-active mihomo
ip rule | grep -E 'sport (22|3389)'
ip route get 1.1.1.1                      # 应显示走 MihomoTUN
ip route get <你的本地客户端IP> sport 22  # 必须显示走主网卡 eth0，绝不能是 MihomoTUN

# 2. 校验配置文件语法
mihomo -t -d /etc/mihomo

# 3. 对比默认出口与专属 ISP 出口的真实 IP / ASN 属性
curl -s [https://ipinfo.io/json](https://ipinfo.io/json)                               # 查看默认机场出口
curl -x socks5h://127.0.0.1:7891 -s [https://ipinfo.io/json](https://ipinfo.io/json)   # 查看 7891 端口静态 ISP 真实落地出口与 ASN
curl -s [https://claude.ai/cdn-cgi/trace](https://claude.ai/cdn-cgi/trace) | grep -E '^(ip|colo|loc)='

# 4. 实时查看 Mihomo 内部真实流量链路（排查分流是否生效的终极手段）
curl -s 127.0.0.1:9090/connections | python3 -c '
import sys, json
for c in json.load(sys.stdin).get("connections", []):
    m = c.get("metadata", {})
    print(m.get("process", "?"), m.get("host") or m.get("destinationIP"), "->", c.get("chains"))'
```

---

## 7. 常见故障速查表

| 故障现象 | 根本原因与修复方案 |
| :--- | :--- |
| **启动 Mihomo 瞬间 SSH 和远程桌面同时卡死断开** | 缺失 `sport 22` / `sport 3389` 策略路由规则。通过云厂商网页 VNC 登录，执行 `/usr/local/bin/protect-ssh.sh`，并检查 `mihomo.service` 中的 `ExecStartPre`。 |
| **登录时报 `Couldn't complete sign-in. Try signing in from a different browser.`** | 1) 出口仍是机房 ASN 而非 ISP；<br>2) 修改了 `--user-agent` 造成平台指纹冲突（请立即移除假 UA）；<br>3) `challenges.cloudflare.com` 未与主站走同一出口；<br>4) Chrome 虚拟显卡被拒 —— **换用原生 `firefox-esr` 并在同一 Firefox 内打开 Gmail 点击登录链接**。 |
| **`dialer-proxy` 不生效或连接静默失败** | 机场订阅节点名称通常含 Emoji 或倍率后缀（如 `🇺🇸 US-01 (0.5x)`），直接写名称极易匹配失败。请用 `select` 代理组包装 `proxy-providers`，再将 `dialer-proxy` 指向该代理组名称。 |
| **挂上 SOCKS5 静态代理后打开目标网页一直转圈超时** | 绝大多数 SOCKS5 住宅代理不支持 UDP（`udp: false`），而浏览器默认发起 QUIC（UDP 443）。必须将 `AND,((NETWORK,udp),(DST-PORT,443)),REJECT` 放在 `rules:` 的**第一行**，逼迫浏览器立即回退到 TCP。 |
| **配置的 SOCKS5 服务器 IP 与 `ipinfo.io` 查出的 IP 不一致** | 正常现象。许多静态住宅代理采用“中转入口 IP + 真实 ISP 落地出口 IP”架构。注意：**系统时区必须与 `ipinfo.io` 查出的最终落地 IP 所在城市对齐**。 |
| **Firefox 启动时弹窗报错 `Profile is already in use`** | 上次异常退出残留了锁文件。执行 `rm -f <profile>/lock <profile>/.parentlock <profile>/.startup-incomplete` 即可。 |

---

## 8. 一键安全回滚（Rollback）

若需彻底停用代理并恢复服务器初始直连状态：

```bash
sudo systemctl disable --now mihomo
sudo ip rule del pref 100 2>/dev/null || true
sudo ip rule del pref 101 2>/dev/null || true
sudo sed -i 's/DNSStubListener=no/DNSStubListener=yes/g' /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
```