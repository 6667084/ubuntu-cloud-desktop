# Ubuntu 24.04 Cloud Desktop - DD 重装完全指南

通过 DD 方式在腾讯云轻量服务器上一键重装 Ubuntu 24.04 LTS Desktop + GNOME (Xorg)，配备 Cloud Kernel、TCP BBR、中文环境、Google Chrome、Node.js，适合需要桌面环境的云服务器场景。

## 适用场景

- 腾讯云轻量应用服务器（Lighthouse）
- 其他支持 DD 重装的 KVM 虚拟化云服务器
- 需要在无桌面的云服务器上运行 GUI 应用（如浏览器自动化、openclaw 等）

## 服务器要求

| 项目 | 最低要求 |
|------|---------|
| CPU | 2 核 |
| 内存 | 4 GB（推荐 8 GB） |
| 磁盘 | 20 GB（推荐 40 GB+） |
| 架构 | x86_64 |
| 虚拟化 | KVM |

## 最终环境

| 组件 | 版本/配置 |
|------|----------|
| OS | Ubuntu 24.04.x LTS (Noble) |
| 内核 | linux-image-virtual-hwe-24.04 (Cloud Kernel) |
| 桌面 | GNOME 46 + Xorg (WaylandEnable=false) |
| 浏览器 | Google Chrome (.deb) |
| 运行时 | Node.js v24.x + npm |
| 输入法 | fcitx5 + 拼音 |
| 网络 | IPv4 DHCP + IPv6 /128 (腾讯云 SDN) |
| TCP | BBR 拥塞控制 |
| 远程桌面 | TigerVNC (端口可自定义) |
| SSH | 非标端口 + 密钥登录 + 禁用密码 |

## 架构概览

```
┌─────────────────────────────────────────────────┐
│                 腾讯云轻量服务器                    │
│                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────┐ │
│  │ SSH         │  │ VNC          │  │ DNS      │ │
│  │ :YOUR_PORT  │  │ :YOUR_PORT   │  │ 内网 DNS │ │
│  │ 密钥认证     │  │ 密码认证      │  │          │ │
│  └──────┬──────┘  └──────┬───────┘  └────┬─────┘ │
│         │                │                │        │
│  ┌──────┴────────────────┴────────────────┴─────┐ │
│  │              netplan (networkd)               │ │
│  │  ens5: DHCP4 + IPv6 static /128              │ │
│  │  IPv6 GW: fe80::feee:ffff:feff:ffff          │ │
│  └──────────────────┬──────────────────────────┘ │
│                     │                             │
│  ┌──────────────────┴──────────────────────────┐ │
│  │           GNOME Desktop + Xorg               │ │
│  │  Chrome | Node.js | fcitx5 | BBR             │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

## 快速开始

> 以下所有 `YOUR_xxx` 占位符请替换为你自己的实际值。

### 1. 准备工作

在本地收集以下信息（从腾讯云控制台和当前系统获取）：

```bash
# 在当前服务器上执行，收集网络参数
ip addr show          # 记录 IPv4 地址、网卡名、MAC
ip -6 addr show       # 记录 IPv6 地址
ip -6 route show      # 记录 IPv6 网关
cat /etc/resolv.conf  # 记录 DNS 服务器
```

需要记录的参数：

| 参数 | 获取方式 | 示例 |
|------|---------|------|
| 公网 IP | 腾讯云控制台 | `YOUR_SERVER_IP` |
| 内网 IP / 子网 | `ip addr show` | `10.x.x.x/22` |
| IPv6 地址 | 腾讯云控制台 → IPv6 | `2402:4e00:...:0/128` |
| IPv6 网关 | `ip -6 route show` | `fe80::feee:ffff:feff:ffff` |
| MAC 地址 | `ip link show` | `52:54:00:xx:xx:xx` |
| DNS | `resolvectl status` | `183.60.83.19` |
| SSH 公钥 | `cat ~/.ssh/authorized_keys` | `ssh-rsa AAAA...` |

### 2. 下载并执行 DD 脚本

```bash
# SSH 到当前服务器，切换 root
sudo -i

# 下载重装脚本（优先使用本仓库镜像，保证可用性）
curl -O https://raw.githubusercontent.com/6667084/ubuntu-cloud-desktop/main/scripts/reinstall.sh
# 备用（原项目最新版）:
# curl -O https://raw.githubusercontent.com/bin456789/reinstall/main/reinstall.sh
# 国内镜像:
# curl -O https://cnb.cool/bin456789/reinstall/-/git/raw/main/reinstall.sh
chmod +x reinstall.sh

# 执行 DD 重装 Ubuntu 24.04
# 替换 YOUR_SSH_PORT 为你想要的 SSH 端口，YOUR_PUBLIC_KEY 为你的公钥
bash reinstall.sh ubuntu 24.04 \
  --ssh-port YOUR_SSH_PORT \
  --ssh-key "YOUR_PUBLIC_KEY"
```

> **注意**：
> - 此操作会擦除整个磁盘，不可恢复
> - 执行后系统自动重启，等待 10-20 分钟
> - 可通过腾讯云控制台 VNC 观察安装进度
> - 安装期间 SSH 端口仍为设定的端口，可观察日志
> - 如需取消（重启前）：`bash reinstall.sh reset`

### 3. 首次连接

```bash
# 等待 10-20 分钟后尝试连接
ssh -i YOUR_PRIVATE_KEY.pem -p YOUR_SSH_PORT root@YOUR_SERVER_IP
```

如果连接成功，说明 DD 安装完成。

### 4. 配置腾讯云内网镜像源

腾讯云内网源 `mirrors.tencentyun.com` 走内网流量，速度快且免费。

```bash
# Ubuntu 24.04 使用 DEB822 格式
cat > /etc/apt/sources.list.d/ubuntu.sources << 'EOF'
Types: deb
URIs: http://mirrors.tencentyun.com/ubuntu
Suites: noble noble-updates noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
EOF

# 清空旧源
> /etc/apt/sources.list
apt update
```

### 5. 修复 DNS（关键步骤）

DD 安装后，cloud-init 配置的 DNS 可能包含不可达的 IPv6 DNS 服务器，导致解析失败。

```bash
# 确认网卡名（可能是 eth0 或 ens5）
ip link show

# 禁用 cloud-init 网络管理
echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

# 写入正确的 netplan 配置
# 注意：根据实际网卡名修改 YOUR_NIC
# YOUR_IPV6_ADDR / YOUR_MAC / YOUR_DNS_1 / YOUR_DNS_2 替换为实际值
cat > /etc/netplan/50-cloud-init.yaml << EOF
network:
    version: 2
    renderer: networkd
    ethernets:
        YOUR_NIC:
            accept-ra: false
            addresses:
            - YOUR_IPV6_ADDR/128
            dhcp4: true
            match:
                macaddress: YOUR_MAC
            nameservers:
                addresses:
                - YOUR_DNS_1
                - YOUR_DNS_2
            routes:
            -   on-link: true
                to: "::/0"
                via: "fe80::feee:ffff:feff:ffff"
EOF
```

> **腾讯云 IPv6 关键参数**：
> - 网关固定为 `fe80::feee:ffff:feff:ffff`（所有实例通用）
> - 地址掩码永远是 `/128`
> - `accept-ra` 必须设为 `false`
> - IPv6 地址必须先在腾讯云控制台开启并获取

### 6. 防止 NetworkManager 冲突

安装 GNOME 后，NetworkManager 会尝试接管网络，导致断网。

```bash
# 禁止 NetworkManager 管理以太网（仅管理 WiFi/WWAN）
mkdir -p /etc/NetworkManager/conf.d
cat > /etc/NetworkManager/conf.d/10-globally-managed-devices.conf << 'EOF'
[keyfile]
unmanaged-devices=*,except:type:wifi,except:type:wwan
EOF

# 应用网络配置
netplan apply
```

### 7. 安装 GNOME Desktop

```bash
# 最小化安装（推荐，节省空间）
DEBIAN_FRONTEND=noninteractive apt install -y ubuntu-desktop-minimal xserver-xorg dbus-x11

# 强制使用 Xorg（Wayland 在远程桌面中兼容性差）
sed -i 's/^#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf

# 设置图形界面启动
systemctl set-default graphical.target
```

### 8. 配置 Swap

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### 9. 安装 Google Chrome

```bash
# Google 下载 CDN（dl.google.com 非 google.com，国内通常可访问）
curl -fsSL -o /tmp/chrome.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
apt install -y /tmp/chrome.deb
rm /tmp/chrome.deb
```

如果 `dl.google.com` 不可达，可用本地代理下载后 scp 上传。

### 10. 安装 Node.js（国内镜像）

```bash
# 从 npmmirror（淘宝镜像）下载二进制包
# 先查最新版本号
NODE_VER=$(curl -s https://registry.npmmirror.com/-/binary/node/latest-v24.x/ | grep -oP 'node-v24\.\d+\.\d+' | head -1)
echo "Installing ${NODE_VER}..."

curl -fsSL "https://registry.npmmirror.com/-/binary/node/${NODE_VER}/${NODE_VER}-linux-x64.tar.xz" \
  | tar -xJ -C /usr/local --strip-components=1

node --version  # 验证
```

### 11. 中文环境 + 输入法

```bash
apt install -y language-pack-zh-hans fonts-noto-cjk fcitx5 fcitx5-chinese-addons
locale-gen zh_CN.UTF-8
update-locale LANG=zh_CN.UTF-8 LC_ALL=zh_CN.UTF-8

# 全局 fcitx5 环境变量
cat > /etc/profile.d/fcitx5.sh << 'EOF'
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export SDL_IM_MODULE=fcitx
export INPUT_METHOD=fcitx
EOF
chmod +x /etc/profile.d/fcitx5.sh
```

### 12. 系统优化（BBR + 内核参数）

```bash
cat > /etc/sysctl.d/99-optimize.conf << 'EOF'
# TCP BBR
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr

# 内存
vm.swappiness=10
vm.dirty_ratio=10
vm.dirty_background_ratio=5

# 网络
net.core.somaxconn=65535
net.ipv4.tcp_tw_reuse=1
net.ipv4.tcp_fin_timeout=15
net.ipv4.tcp_keepalive_time=600
net.ipv4.conf.all.send_redirects=0
net.ipv4.conf.default.accept_redirects=0

# 安全
kernel.dmesg_restrict=1
kernel.printk=3 3 1 3
EOF

# IPv6 accept_ra 必须关闭（腾讯云 SDN）
# 替换 YOUR_NIC 为实际网卡名
cat > /etc/sysctl.d/99-ipv6.conf << 'EOF'
net.ipv6.conf.YOUR_NIC.accept_ra=0
net.ipv6.conf.all.accept_ra=0
net.ipv6.conf.default.accept_ra=0
EOF

sysctl --system
```

### 13. SSH 加固

```bash
# 注意：必须同时检查 /etc/ssh/sshd_config.d/ 下所有文件
cat > /etc/ssh/sshd_config.d/hardened.conf << 'EOF'
Port YOUR_SSH_PORT
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
EOF

# 删除脚本自动生成的端口配置（如有冲突）
rm -f /etc/ssh/sshd_config.d/01-change-ssh-port.conf

systemctl restart ssh  # Ubuntu 24.04 服务名是 ssh 不是 sshd
```

### 14. 防火墙

```bash
ufw allow YOUR_SSH_PORT/tcp    # SSH
ufw allow YOUR_VNC_PORT/tcp    # VNC
echo "y" | ufw enable
```

> **同时需要在腾讯云控制台防火墙中放行对应端口。**

### 15. 配置 VNC 远程桌面

```bash
# 安装 TigerVNC
apt install -y tigervnc-standalone-server tigervnc-common dbus-x11

# 为桌面用户配置 VNC（假设用户名为 YOUR_DESKTOP_USER）
# 设置 VNC 密码（与系统密码无关）
su - YOUR_DESKTOP_USER -c 'mkdir -p ~/.vnc'
su - YOUR_DESKTOP_USER -c 'echo "YOUR_VNC_PASSWORD" | vncpasswd -f > ~/.vnc/passwd && chmod 600 ~/.vnc/passwd'

# xstartup（关键：必须用 dbus-launch 包装 gnome-session）
su - YOUR_DESKTOP_USER -c 'cat > ~/.vnc/xstartup << XEOF
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
export XDG_SESSION_TYPE=x11
export GDK_BACKEND=x11
export XDG_SESSION_DESKTOP=ubuntu
export XDG_CURRENT_DESKTOP=ubuntu:GNOME
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export LANG=zh_CN.UTF-8
export LC_ALL=zh_CN.UTF-8
exec dbus-launch --exit-with-session gnome-session
XEOF
chmod +x ~/.vnc/xstartup'

# VNC 配置
su - YOUR_DESKTOP_USER -c 'cat > ~/.vnc/config << EOF
geometry=1920x1080
depth=24
rfbport=YOUR_VNC_PORT
localhost=no
SecurityTypes=VncAuth
EOF'

# 修复 hostname 解析
grep "$(hostname)" /etc/hosts || echo "127.0.1.1 $(hostname)" >> /etc/hosts

# 创建 systemd 开机自启服务
cat > /etc/systemd/system/vncserver.service << 'EOF'
[Unit]
Description=TigerVNC Remote Desktop
After=network.target

[Service]
Type=forking
User=YOUR_DESKTOP_USER
ExecStartPre=-/usr/bin/vncserver -kill :2
ExecStart=/usr/bin/vncserver :2
ExecStop=/usr/bin/vncserver -kill :2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable vncserver.service
systemctl start vncserver.service
```

### 16. 禁用 cloud-init

```bash
touch /etc/cloud/cloud-init.disabled
```

### 17. 验证清单

```bash
echo "=== System ==="
echo "OS: $(lsb_release -ds)"
echo "Kernel: $(uname -r)"
echo "Cloud Kernel: $(dpkg -l linux-image-virtual-hwe-24.04 2>/dev/null | tail -1 | awk '{print $3}')"
echo "Timezone: $(timedatectl show -p Timezone --value)"

echo "=== Network ==="
echo "IPv4: $(ip -4 addr show | grep 'scope global' | awk '{print $2}')"
echo "IPv6: $(ip -6 addr show | grep 'scope global' | awk '{print $2}')"
echo "DNS: $(resolvectl status 2>/dev/null | grep 'DNS Servers' | head -1 | awk -F: '{print $2}' | xargs)"
echo "Mirror: $(grep URIs /etc/apt/sources.list.d/ubuntu.sources 2>/dev/null | awk '{print $2}')"
echo "BBR: $(sysctl -n net.ipv4.tcp_congestion_control) / $(sysctl -n net.core.default_qdisc)"

echo "=== Services ==="
echo "SSH: $(systemctl is-active ssh)"
echo "VNC: $(ss -tlnp | grep -q 'vnc' && echo 'running' || echo 'NOT running')"
echo "Firewall: $(systemctl is-active ufw)"
echo "Cloud-init: $(test -f /etc/cloud/cloud-init.disabled && echo 'disabled' || echo 'ACTIVE')"

echo "=== Applications ==="
echo "Desktop: $(dpkg -l ubuntu-desktop-minimal 2>/dev/null | tail -1 | awk '{print $3}')"
echo "Chrome: $(google-chrome --version 2>/dev/null || echo 'NOT installed')"
echo "Node: $(node --version 2>/dev/null || echo 'NOT installed')"
echo "Swap: $(free -h | grep -i swap | awk '{print $2}')"
```

## 腾讯云 IPv6 特殊说明

腾讯云轻量服务器的 IPv6 实现与标准网络不同：

```
┌──────────────────────────────────────────────┐
│            腾讯云 SDN 网络架构                  │
│                                                │
│   VM (ens5) ─── 虚拟交换机 ─── SDN 网关         │
│       │              │              │          │
│   IPv4: DHCP      MAC: fe:ee:xx    IPv4 GW     │
│   IPv6: /128      (共用 MAC)      IPv6 GW      │
│                   │              (固定)        │
│                   │              fe80::feee:    │
│                   │              ffff:feff:ffff │
│                                                │
│   关键规则:                                      │
│   1. 地址掩码永远 /128（没有子网）                 │
│   2. 网关固定 fe80::feee:ffff:feff:ffff         │
│   3. accept_ra 必须 = 0（不用 SLAAC）            │
│   4. IPv4/IPv6 网关共用同一 MAC                   │
│   5. 地址必须先在控制台分配，SDN 才会路由          │
└──────────────────────────────────────────────┘
```

**为什么手动配的 IPv6 不通？**

SDN（软件定义网络）要求 IPv6 地址必须先在腾讯云控制台分配过，虚拟交换机才会转发该地址的数据包。未在控制台注册的地址会被直接丢弃。

## 踩坑记录

### 1. NetworkManager 接管导致断网

**现象**：安装 `ubuntu-desktop-minimal` 后，NetworkManager 作为依赖被安装，netplan 渲染器自动从 `networkd` 切换为 `NetworkManager`，但 NM 未正确配置，网卡变为 `unmanaged` 状态，丢失 IP。

**解决**：在 netplan 中显式指定 `renderer: networkd`，并通过 NM 配置文件禁止其管理以太网设备。

### 2. VNC gnome-session 秒退

**现象**：`vncserver -list` 显示会话存在，但连接后黑屏或立即断开。

**原因**：xstartup 中直接 `exec gnome-session` 缺少 D-Bus 会话。

**解决**：使用 `exec dbus-launch --exit-with-session gnome-session`，并安装 `dbus-x11` 包。

### 3. DNS 解析失败（IPv6 DNS 优先）

**现象**：`apt update` 报 `No address associated with hostname`，但手动指定 DNS 查询正常。

**原因**：cloud-init 配置了 IPv6 DNS 服务器，systemd-resolved 优先使用不可达的 IPv6 DNS。

**解决**：在 netplan 的 nameservers 中只配置 IPv4 DNS，并禁用 cloud-init 网络管理。

### 4. hostname 解析失败

**现象**：`vncserver` 报 `Could not acquire fully qualified host name`。

**解决**：`echo "127.0.1.1 $(hostname)" >> /etc/hosts`

## macOS 连接 VNC

```bash
# Finder → 前往 → 连接服务器
open vnc://YOUR_SERVER_IP:YOUR_VNC_PORT
```

或使用第三方 VNC 客户端（如 [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer/)）。

## 参考资源

- [bin456789/reinstall](https://github.com/bin456789/reinstall) - DD 重装脚本（本仓库 scripts/ 下有镜像备份）
- [TigerVNC](https://github.com/TigerVNC/tigervnc) - VNC 服务器
- [Ubuntu Cloud Images](https://cloud-images.ubuntu.com/) - Ubuntu 官方云镜像
- [npmmirror](https://npmmirror.com/) - Node.js 国内镜像

## License

MIT
