# 主流网络代理与隧道协议详解

## 1. OpenVPN

### 1.1 简介

OpenVPN 是一款开源的 VPN（虚拟专用网络）软件，由 James Yonan 于 2001 年创建，基于 **OpenSSL 库**和 **TLS 协议**实现，是目前应用最广泛的 VPN 解决方案之一。

### 1.2 工作原理

OpenVPN 通过在客户端和服务端之间建立一条加密的虚拟隧道来传输数据：

```
客户端 → TLS握手 → 服务端
         ↓
     建立虚拟网卡 (tun/tap)
         ↓
   所有流量经加密隧道传输
```

- **tun 模式**：工作在第三层（网络层），用于路由 IP 数据包
- **tap 模式**：工作在第二层（数据链路层），用于桥接网络

### 1.3 核心特性

| 特性     | 说明                                |
| -------- | ----------------------------------- |
| 协议支持 | UDP / TCP（推荐 UDP，速度更快）     |
| 加密算法 | AES-256-GCM、ChaCha20 等            |
| 认证方式 | 证书（PKI）、用户名密码、预共享密钥 |
| 跨平台   | Windows、macOS、Linux、Android、iOS |
| 端口     | 默认 1194（UDP），可自定义          |

### 1.4 优点

- **安全性高**：基于 TLS，使用成熟的加密体系，经过大量安全审计
- **高度可配置**：支持多种加密方式、认证方法和路由策略
- **社区成熟**：大量文档、教程和商业支持
- **穿透性好**：支持 TCP 443 端口，可伪装成 HTTPS 流量

### 1.5 缺点

- **速度相对较慢**：协议头部较重，TLS 握手开销大
- **配置复杂**：需要搭建 PKI 体系，配置文件较复杂
- **特征明显**：流量特征较明显，容易被深度包检测（DPI）识别
- **移动端体验一般**：频繁切换网络时重连较慢

### 1.6 典型应用场景

- 企业内网远程接入（Site-to-Site VPN）
- 团队协作安全通信
- 服务器集群内网互联
- 个人隐私保护（需配合混淆）

### 1.7 快速部署示例

**服务端安装（Ubuntu）：**

```bash
apt update && apt install openvpn easy-rsa -y

# 初始化 PKI
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
```

**客户端配置文件示例（client.ovpn）：**

```ini
client
dev tun
proto udp
remote your-server-ip 1194
resolv-retry infinite
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
cipher AES-256-GCM
auth SHA256
verb 3
```

---

## 2. Shadowsocks

### 2.1 简介

Shadowsocks（SS）是由中国开发者 clowwindy 于 2012 年创建的一种**加密代理协议**，专为穿透网络审查而设计。其核心思想是将流量伪装成普通的加密数据，难以被特征检测。

### 2.2 工作原理

```
本地应用 → SOCKS5本地代理 → 加密 → 服务端 → 目标网站
                ↑
          ss-local (客户端)
                              ↑
                        ss-server (服务端)
```

Shadowsocks 使用**对称加密**对流量进行加密，不建立完整的 VPN 隧道，而是作为 SOCKS5 代理工作，只代理指定应用的流量。

### 2.3 核心特性

| 特性     | 说明                                         |
| -------- | -------------------------------------------- |
| 协议类型 | SOCKS5 代理（非全局 VPN）                    |
| 加密方式 | AEAD（推荐）：AES-256-GCM、ChaCha20-Poly1305 |
| 传输层   | TCP / UDP                                    |
| 混淆     | 支持插件（obfs、v2ray-plugin 等）            |
| 实现语言 | Python（原版）、C、Go、Rust 等多种实现       |

### 2.4 主要实现版本

```
Shadowsocks-Python    → 原版，已停止维护
Shadowsocks-libev     → C 语言实现，轻量高效，广泛使用
Shadowsocks-rust      → Rust 实现，性能最优
Shadowsocks-go        → Go 实现
```

### 2.5 AEAD 加密详解

现代 Shadowsocks 推荐使用 AEAD（Authenticated Encryption with Associated Data）加密：

```
数据包结构：
[加密后的目标地址][加密后的数据块1][加密后的数据块2]...
       ↑                ↑
   包含认证标签      包含认证标签
```

AEAD 同时提供**加密**和**完整性验证**，防止主动探测攻击。

### 2.6 优点

- **轻量简洁**：协议简单，资源占用少
- **流量混淆好**：流量看起来像随机数据，难以识别
- **配置简单**：只需服务器地址、端口、密码、加密方式四个参数
- **性能优秀**：尤其是 libev 和 rust 实现

### 2.7 缺点

- **非全局代理**：只代理配置的应用，需要客户端支持
- **无流量伪装**：虽然加密，但没有伪装成正常协议的能力（需插件）
- **抗主动探测弱**（旧版本）：需要 AEAD 模式才能防御主动探测
- **原项目停止维护**：原作者迫于压力删库，社区分叉维护

### 2.8 配置示例

**服务端配置（config.json）：**

```json
{
  "server": "0.0.0.0",
  "server_port": 8388,
  "password": "your-strong-password",
  "timeout": 300,
  "method": "chacha20-ietf-poly1305",
  "fast_open": true,
  "nameserver": "8.8.8.8",
  "mode": "tcp_and_udp"
}
```

**启动服务：**

```bash
# 使用 shadowsocks-libev
ss-server -c /etc/shadowsocks/config.json -d start

# 使用 Docker
docker run -d \
  -p 8388:8388/tcp \
  -p 8388:8388/udp \
  shadowsocks/shadowsocks-libev \
  ss-server -s 0.0.0.0 -p 8388 -k password -m chacha20-ietf-poly1305
```

### 2.9 混淆插件

为进一步隐藏流量特征，可配合插件使用：

```bash
# simple-obfs：将流量伪装成 HTTP/HTTPS
obfs=http
obfs-host=www.bing.com

# v2ray-plugin：将流量伪装成 WebSocket+TLS
plugin=v2ray-plugin
plugin-opts=server;tls;host=your-domain.com
```

---

## 3. V2Ray

### 3.1 简介

V2Ray 是由 Project V 团队开发的一款**网络代理框架**，于 2015 年发布。与 Shadowsocks 相比，V2Ray 更像是一个平台，支持多种协议和传输方式的组合，灵活性极强。其主要实现为 **Xray-core**（V2Ray 的超集，性能更优）。

### 3.2 架构设计

V2Ray 采用模块化设计：

```
┌─────────────────────────────────────────┐
│                V2Ray Core               │
├──────────┬──────────────┬───────────────┤
│  Inbound │    Router    │   Outbound    │
│  入站协议 │   路由分流   │   出站协议    │
├──────────┴──────────────┴───────────────┤
│            Transport Layer              │
│  传输层：TCP/WebSocket/gRPC/HTTP2/QUIC  │
├─────────────────────────────────────────┤
│         Security Layer (TLS/XTLS)       │
└─────────────────────────────────────────┘
```

### 3.3 支持的协议

**入站/出站协议：**

| 协议          | 说明                                   |
| ------------- | -------------------------------------- |
| VMess         | V2Ray 自研协议，加密+认证              |
| VLESS         | VMess 精简版，去掉内置加密（配合 TLS） |
| Trojan        | 支持 Trojan 协议                       |
| Shadowsocks   | 兼容 SS 协议                           |
| SOCKS         | 标准 SOCKS4/5                          |
| HTTP          | HTTP 代理                              |
| Dokodemo-door | 任意门，透明代理                       |

**传输方式（Transport）：**

| 传输方式  | 说明                     |
| --------- | ------------------------ |
| TCP       | 原始 TCP                 |
| WebSocket | 伪装 WS，可配合 CDN      |
| HTTP/2    | 复用连接，性能好         |
| gRPC      | Google RPC 协议          |
| QUIC      | 基于 UDP，低延迟         |
| mKCP      | 魔改 KCP，低延迟但流量大 |

### 3.4 VMess 协议详解

VMess 是 V2Ray 的核心协议：

```
请求格式：
┌──────────┬─────────┬──────────────┬──────────┐
│  认证信息 │  指令块  │   数据块头   │  数据内容 │
│  (16字节) │ (加密)  │   (加密)     │  (加密)  │
└──────────┴─────────┴──────────────┴──────────┘

认证信息 = HMAC(MD5(用户ID + 时间戳))
```

- 使用 **UUID** 作为用户标识
- 时间戳参与认证，防止重放攻击（允许±90秒误差）
- 支持动态端口（alterId，新版已弃用）

### 3.5 路由功能

V2Ray 强大的路由分流是其核心优势：

```json
{
  "routing": {
    "rules": [
      {
        "type": "field",
        "domain": ["geosite:cn"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "ip": ["geoip:cn", "geoip:private"],
        "outboundTag": "direct"
      },
      {
        "type": "field",
        "domain": ["geosite:ads"],
        "outboundTag": "block"
      }
    ]
  }
}
```

实现**国内直连、国外代理、广告屏蔽**三合一。

### 3.6 完整配置示例

**服务端配置（config.json）：**

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [{ "id": "your-uuid-here", "flow": "xtls-rprx-vision" }],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "certificates": [
            {
              "certificateFile": "/etc/ssl/cert.pem",
              "keyFile": "/etc/ssl/key.pem"
            }
          ]
        }
      }
    }
  ],
  "outbounds": [{ "protocol": "freedom" }]
}
```

**客户端配置（带路由分流）：**

```json
{
  "inbounds": [
    {
      "port": 1080,
      "protocol": "socks",
      "settings": { "auth": "noauth", "udp": true }
    }
  ],
  "outbounds": [
    {
      "tag": "proxy",
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "your-server.com",
            "port": 443,
            "users": [
              {
                "id": "your-uuid",
                "flow": "xtls-rprx-vision",
                "encryption": "none"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls"
      }
    },
    { "tag": "direct", "protocol": "freedom" },
    { "tag": "block", "protocol": "blackhole" }
  ],
  "routing": {
    "rules": [
      { "type": "field", "domain": ["geosite:cn"], "outboundTag": "direct" },
      { "type": "field", "ip": ["geoip:cn"], "outboundTag": "direct" },
      { "type": "field", "outboundTag": "proxy" }
    ]
  }
}
```

### 3.7 XRAY 与 XTLS

**Xray-core** 是 V2Ray 的分叉，新增了：

- **XTLS**：在 TLS 内层直接传输 TLS 数据，减少重复加密，性能提升 30-50%
- **VLESS + XTLS Vision**：目前最推荐的高性能搭配
- **Reality 协议**：无需域名证书，直接借用真实网站的 TLS 指纹

```
传统方案：应用数据 → TLS(V2Ray) → TLS(外层) → 两次加密
XTLS 方案：应用数据 → TLS(直通) → 只有一层加密，性能翻倍
```

### 3.8 优点

- **极高灵活性**：协议、传输方式可自由组合
- **强大路由**：精细化分流，支持 geodata
- **性能优秀**：XTLS/VLESS 性能媲美原生 TLS
- **伪装能力强**：可伪装成 WebSocket、gRPC 等主流协议
- **CDN 中转**：WebSocket 模式支持 Cloudflare CDN 中转

### 3.9 缺点

- **配置复杂**：JSON 配置文件较长，学习曲线陡
- **协议迭代快**：VMess → VLESS → Reality，需要跟进更新
- **资源占用**：相比 Shadowsocks 略重

---

## 4. Trojan

### 4.1 简介

Trojan 是一种**将代理流量伪装成正常 HTTPS 流量**的协议，名字来源于"特洛伊木马"。其核心思想是：**如果代理流量与正常 HTTPS 流量无法区分，那么它就无法被识别和屏蔽**。

### 4.2 工作原理

Trojan 的设计极为精妙：

```
                    ┌─────────────────────────────┐
                    │         Trojan 服务端        │
合法 HTTPS 请求 ───→ │                             │
                    │  密码正确？                  │
代理请求 ─────────→ │   ├─ 是 → 转发代理流量       │
                    │   └─ 否 → 转发给真实 Web 服务 │
                    └─────────────────────────────┘
                                  ↑
                          监听 443 端口（真实 TLS）
```

**关键设计：**

1. 监听标准 HTTPS 端口（443）
2. 使用**真实的 TLS 证书**（Let's Encrypt 等）
3. 流量格式与 HTTPS 完全相同
4. 密码错误时，**将请求转发给真实的 Web 服务器**，使探测者看到正常网站

### 4.3 Trojan 协议格式

```
Trojan 请求格式（TLS 解密后）：

┌──────────────┬──────┬─────────────┬──────┬──────────┐
│  SHA224(密码) │ CRLF │  SOCKS5地址 │ CRLF │  数据内容 │
│   (56字节)   │      │             │      │          │
└──────────────┴──────┴─────────────┴──────┴──────────┘
```

密码使用 SHA224 哈希传输，服务端通过比对哈希值验证身份。

### 4.4 Trojan-GFW vs Trojan-Go

| 特性      | Trojan-GFW（原版） | Trojan-Go           |
| --------- | ------------------ | ------------------- |
| 语言      | C++                | Go                  |
| 多路复用  | 不支持             | 支持（smux）        |
| WebSocket | 不支持             | 支持（CDN中转）     |
| 路由      | 不支持             | 支持                |
| AEAD 加密 | 不支持             | 支持（obfuscation） |
| 性能      | 高                 | 较高                |
| 维护状态  | 停止维护           | 活跃                |

### 4.5 配置示例

**服务端配置（config.json）：**

```json
{
  "run_type": "server",
  "local_addr": "0.0.0.0",
  "local_port": 443,
  "remote_addr": "127.0.0.1",
  "remote_port": 80,
  "password": ["your-strong-password"],
  "ssl": {
    "cert": "/etc/letsencrypt/live/your-domain.com/fullchain.pem",
    "key": "/etc/letsencrypt/live/your-domain.com/privkey.pem",
    "alpn": ["h2", "http/1.1"]
  },
  "tcp": {
    "no_delay": true,
    "keep_alive": true,
    "fast_open": false
  },
  "log_level": 1
}
```

**客户端配置：**

```json
{
  "run_type": "client",
  "local_addr": "127.0.0.1",
  "local_port": 1080,
  "remote_addr": "your-domain.com",
  "remote_port": 443,
  "password": ["your-strong-password"],
  "ssl": {
    "verify": true,
    "verify_hostname": true,
    "sni": "your-domain.com"
  }
}
```

**Nginx 后端配置（处理非 Trojan 请求）：**

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

### 4.6 Trojan-Go 配置（支持 WebSocket）

```json
{
  "run_type": "server",
  "local_addr": "0.0.0.0",
  "local_port": 443,
  "remote_addr": "127.0.0.1",
  "remote_port": 80,
  "password": ["your-password"],
  "ssl": {
    "cert": "/path/to/cert.pem",
    "key": "/path/to/key.pem"
  },
  "websocket": {
    "enabled": true,
    "path": "/ws",
    "host": "your-domain.com"
  },
  "mux": {
    "enabled": true,
    "concurrency": 8,
    "idle_timeout": 60
  }
}
```

### 4.7 优点

- **伪装能力最强**：与真实 HTTPS 流量完全相同，极难被 DPI 识别
- **主动探测防御**：密码错误时回退到真实网站，探测者看到正常页面
- **配置相对简单**：比 V2Ray 更简洁
- **无需混淆插件**：本身就是 TLS，无需额外混淆

### 4.8 缺点

- **必须有域名和证书**：部署成本略高
- **端口固定**：通常必须使用 443 端口
- **路由功能弱**（原版）：需要 Trojan-Go 或配合客户端实现分流
- **多用户管理复杂**：不如 V2Ray 灵活

---

## 5. WireGuard

### 5.1 简介

WireGuard 是由 Jason Donenfeld 于 2016 年发布的新一代 VPN 协议，代码极简（仅约 **4000 行**，OpenVPN 超过 70000 行），已于 2020 年合并进 Linux 5.6 内核主线。

### 5.2 核心设计

WireGuard 只使用一套固定的现代密码学算法，无需协商：

```
密钥交换：Curve25519（ECDH）
对称加密：ChaCha20
消息认证：Poly1305
哈希函数：BLAKE2s
握手协议：Noise_IKpsk2
```

没有算法选项，没有版本协商，攻击面极小。

### 5.3 工作原理

```
每个 Peer 拥有一对密钥（公钥/私钥）

握手过程（1-RTT）：
客户端 → 发起握手（携带公钥）→ 服务端
         ← 响应握手             ←
         → 开始传输数据         →

之后所有数据用会话密钥加密，定期轮换（每 3 分钟或 180 秒）
```

### 5.4 对比 OpenVPN

| 项目         | WireGuard      | OpenVPN                |
| ------------ | -------------- | ---------------------- |
| 代码行数     | ~4,000         | ~70,000                |
| 握手延迟     | 1-RTT          | 多次 RTT               |
| 性能         | 极高（内核态） | 中等（用户态）         |
| 移动网络切换 | 无感切换       | 需重连                 |
| 加密算法     | 固定现代算法   | 可配置（也可选弱算法） |
| 流量特征     | UDP，特征明显  | 可配置                 |

### 5.5 配置示例

**服务端（/etc/wireguard/wg0.conf）：**

```ini
[Interface]
PrivateKey = <服务端私钥>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <客户端公钥>
AllowedIPs = 10.0.0.2/32
```

**客户端：**

```ini
[Interface]
PrivateKey = <客户端私钥>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <服务端公钥>
Endpoint = your-server.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

**一键生成密钥：**

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

### 5.6 优缺点

**优点：**

- 性能最优（运行在内核态，吞吐量极高）
- 配置极简，部署快
- 移动端体验好（网络切换无感）
- 代码量少，安全审计容易

**缺点：**

- UDP 协议，流量特征明显，容易被封
- 无内置混淆，不适合翻墙场景（需配合 wstunnel 等工具）
- 默认不支持动态 IP 客户端（需要额外配置）

---

## 6. Hysteria / Hysteria2

### 6.1 简介

Hysteria 是基于 **QUIC 协议**构建的高性能代理，专为**高丢包、高延迟**网络环境（如国际链路）设计，通过激进的拥塞控制算法（BBR 变体）在恶劣网络下仍能保持高速。

### 6.2 工作原理

```
客户端 ←→ QUIC（UDP）←→ 服务端

QUIC 特性：
- 基于 UDP，减少 TCP 队头阻塞
- 0-RTT 快速连接
- 内置 TLS 1.3
- 多路复用（无队头阻塞）
```

Hysteria2 使用类似 Salamander 的混淆，将 QUIC 流量伪装成普通 UDP。

### 6.3 配置示例

**服务端（config.yaml）：**

```yaml
listen: :443

tls:
  cert: /path/to/cert.pem
  key: /path/to/key.pem

auth:
  type: password
  password: your-password

masquerade:
  type: proxy
  proxy:
    url: https://www.bing.com
    rewriteHost: true

bandwidth:
  up: 1 gbps
  down: 1 gbps
```

**客户端：**

```yaml
server: your-server.com:443

auth: your-password

tls:
  sni: your-server.com

bandwidth:
  up: 100 mbps
  down: 300 mbps

socks5:
  listen: 127.0.0.1:1080

http:
  listen: 127.0.0.1:8080
```

### 6.4 适用场景

- 跨大西洋/太平洋的高延迟链路
- 丢包率高的移动网络
- 视频流媒体、大文件传输

---

## 7. TUIC

### 7.1 简介

TUIC（Delicately-TUICed）同样基于 **QUIC 协议**，由中国开发者 EAimTY 开发，专注于**最小化延迟**，支持 0-RTT UDP 代理。

### 7.2 特点

```
协议栈：QUIC + TLS 1.3
代理模式：支持 TCP 和 UDP（0-RTT UDP 转发）
多路复用：原生 QUIC 流复用
拥塞控制：BBR / CUBIC 可选
```

与 Hysteria 的区别：

- Hysteria 追求**高带宽**（激进带宽占用）
- TUIC 追求**低延迟**（保守带宽，延迟优先）

### 7.3 配置示例

```yaml
# 服务端
server: "[::]:443"
users:
  your-uuid: your-password
certificate: /path/to/cert.pem
private_key: /path/to/key.pem
congestion_control: bbr

# 客户端
server: "your-server.com:443"
uuid: your-uuid
password: your-password
socks5_address: "127.0.0.1:1080"
```

---

## 8. NaïveProxy

### 8.1 简介

NaïveProxy 由 klzgrad 开发，将代理流量伪装成**真实的 Chrome 浏览器 HTTP/2 流量**，使用 Chromium 网络栈，TLS 指纹与 Chrome 完全一致。

### 8.2 工作原理

```
客户端（Chromium网络栈）→ HTTP/2 CONNECT → Caddy服务端
                              ↑
                    TLS指纹与真实Chrome相同
                    流量模式与真实HTTP/2相同
```

**核心优势：** 不仅加密内容相同，连 TLS 指纹、HTTP/2 帧格式都与真实 Chrome 一致，极难被机器学习流量分类器识别。

### 8.3 部署（配合 Caddy）

```
// Caddyfile
{
  servers {
    protocol {
      experimental_http3
    }
  }
}

your-domain.com {
  forward_proxy {
    basic_auth user password
    hide_ip
    hide_via
    probe_resistance secret-path.com
  }
  reverse_proxy https://www.example.com {
    header_up Host {upstream_hostport}
  }
}
```

---

## 9. SSH 隧道

### 9.1 简介

SSH（Secure Shell）隧道是最古老的加密代理方案之一，利用 SSH 协议本身的加密能力转发流量，无需额外软件，几乎所有服务器都支持。

### 9.2 三种隧道模式

**本地端口转发（-L）：**

```bash
# 将本地 8080 流量通过 SSH 转发到目标
ssh -L 8080:target-host:80 user@ssh-server

# 示例：访问内网数据库
ssh -L 3306:db.internal:3306 user@bastion-host
```

**远程端口转发（-R）：**

```bash
# 将服务端的 8080 端口转发到本地服务
ssh -R 8080:localhost:3000 user@ssh-server

# 常用于内网穿透
```

**动态端口转发（-D）：SOCKS5 代理**

```bash
# 在本地开启 SOCKS5 代理，所有流量走 SSH 隧道
ssh -D 1080 -N -f user@ssh-server

# 配合 proxychains 使用
proxychains curl https://example.com
```

### 9.3 持久化配置（~/.ssh/config）

```
Host my-proxy
    HostName your-server.com
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    DynamicForward 1080
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ExitOnForwardFailure yes
```

### 9.4 优缺点

**优点：**

- 零配置，服务器开箱即用
- 非常成熟稳定
- 支持公钥认证，安全性高

**缺点：**

- SSH 流量特征明显，容易被识别
- 性能一般
- 不适合移动端长期使用

---

## 10. Tor（洋葱路由）

### 10.1 简介

Tor（The Onion Router）是一个**匿名通信网络**，通过多跳加密路由实现流量匿名化，由非营利组织 Tor Project 维护。

### 10.2 工作原理

```
你的设备 → 入口节点(Guard) → 中间节点(Middle) → 出口节点(Exit) → 目标网站
           ↑                  ↑                   ↑
         第3层加密           第2层加密            第1层加密

每一跳只知道"上一跳"和"下一跳"，无法知道完整路径
```

**洋葱加密：**

```
原始数据
  → 用出口节点公钥加密
    → 用中间节点公钥加密
      → 用入口节点公钥加密
        → 发送（每过一跳，剥掉一层加密）
```

### 10.3 隐藏服务（.onion）

Tor 支持在网络内部运行服务，无需暴露服务器真实 IP：

```bash
# torrc 配置
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:8080

# 启动后在 HiddenServiceDir 下生成 .onion 地址
cat /var/lib/tor/hidden_service/hostname
# → xxxxxxxxxxxxxxxx.onion
```

### 10.4 优缺点

**优点：**

- 匿名性极强，无单点故障
- 分布式网络，无中心服务器

**缺点：**

- 速度很慢（通常 1-10 Mbps）
- 出口节点可能被监控
- 延迟高（通常 200-500ms）

---

## 11. Outline / Outline VPN

Outline 是由 Google Jigsaw 团队基于 Shadowsocks 开发的开源 VPN 工具，提供图形化管理界面：

```
特点：
- 基于 Shadowsocks 协议（chacha20-ietf-poly1305）
- 一键部署脚本，傻瓜式安装
- Web 管理面板，可视化添加/删除用户
- 官方客户端支持全平台
```

```bash
# 一键安装服务端
bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

---

## 12. 综合对比与选型

### 12.1 性能与安全对比

| 方案        | 速度  | 延迟  | 匿名性 | 抗封锁 | 部署难度 |
| ----------- | ----- | ----- | ------ | ------ | -------- |
| WireGuard   | ★★★★★ | ★★★★★ | ★★★    | ★★     | ★★★★★    |
| Hysteria2   | ★★★★★ | ★★★★  | ★★★    | ★★★★   | ★★★★     |
| OpenVPN     | ★★★   | ★★★   | ★★★    | ★★     | ★★       |
| Shadowsocks | ★★★★  | ★★★★  | ★★★    | ★★★    | ★★★★★    |
| V2Ray/Xray  | ★★★★  | ★★★★  | ★★★    | ★★★★★  | ★★       |
| Trojan      | ★★★★  | ★★★★  | ★★★    | ★★★★★  | ★★★      |
| NaïveProxy  | ★★★★  | ★★★★  | ★★★    | ★★★★★  | ★★★      |
| SSH 隧道    | ★★    | ★★★   | ★★     | ★★     | ★★★★★    |
| Tor         | ★     | ★     | ★★★★★  | ★★★    | ★★★★     |
| TUIC        | ★★★★  | ★★★★★ | ★★★    | ★★★★   | ★★★★     |

### 12.2 使用场景选型

```
场景                          推荐方案
──────────────────────────────────────────────────
企业内网 VPN                   WireGuard / OpenVPN
高延迟/丢包网络                Hysteria2
极致匿名                       Tor
流量伪装要求极高                NaïveProxy / Trojan
灵活路由 + 高性能              Xray (VLESS+Reality)
快速搭建个人代理                Outline / Shadowsocks
零服务器成本应急               SSH 隧道
游戏低延迟                     TUIC / WireGuard
```

### 12.3 底层加密算法总结

不论哪种方案，现代安全方案都收敛到以下几个核心算法：

```
密钥交换：
  Curve25519 (X25519)    ← 主流，WireGuard/TLS 1.3
  ECDH P-256             ← 兼容性好

对称加密：
  AES-256-GCM            ← 硬件加速，x86 首选
  ChaCha20-Poly1305      ← 软件实现，移动端首选

哈希/MAC：
  SHA-256 / SHA-384
  BLAKE2s / BLAKE3       ← 更快

握手协议：
  TLS 1.3                ← 最广泛
  Noise Protocol         ← WireGuard 使用
  QUIC（内置 TLS 1.3）   ← Hysteria/TUIC 使用
```

### 12.4 新兴方案一览

| 方案               | 特点                                 | 状态       |
| ------------------ | ------------------------------------ | ---------- |
| **Reality** (Xray) | 借用真实网站 TLS 指纹，无需证书      | 活跃       |
| **VLESS+Vision**   | XTLS 直通，性能极高                  | 活跃       |
| **Brook**          | 跨平台，支持 WebSocket/QUIC          | 活跃       |
| **Clash.Meta**     | 多协议客户端框架，规则强大           | 活跃       |
| **sing-box**       | 下一代统一代理平台，支持所有主流协议 | 活跃，推荐 |

**sing-box** 目前是社区最推荐的全能方案，单一程序支持：

```
Shadowsocks / VMess / VLESS / Trojan / Hysteria2 / TUIC / NaïveProxy / WireGuard ...
```

### 12.5 现代推荐搭配

目前（2025）社区推荐的高安全性方案：

**方案一：VLESS + XTLS-Reality（无需域名）**

```
Xray 服务端 + VLESS协议 + Reality（借用真实TLS指纹）
优点：无需域名证书，抗主动探测，性能极高
```

**方案二：Trojan-Go + WebSocket + TLS + CDN**

```
Trojan-Go → Nginx → Cloudflare CDN
优点：隐藏真实服务器 IP，抗封锁能力强
```

**方案三：Shadowsocks + v2ray-plugin（WebSocket+TLS）**

```
ss-rust + v2ray-plugin（tls模式）+ Cloudflare CDN
优点：配置简单，有 CDN 保护，生态工具丰富
```

---

> **免责声明**：本文仅供技术学习和研究使用，请遵守所在地区相关法律法规，合法合规地使用网络工具。
