# FlClash Docker 部署与使用教程 🚀

基于 Clash Meta 内核的多协议代理客户端 —— 支持 Web 控制面板，适合自建节点管理。

## 目录

- [什么是 FlClash？](#什么是-flclash)
- [Docker 部署](#docker-部署)
  - [快速启动](#快速启动)
  - [Docker Compose](#docker-compose)
- [配置订阅](#配置订阅)
  - [远程订阅 URL](#1-远程订阅-url)
  - [本地配置文件](#2-本地配置文件)
  - [手动添加节点](#3-手动添加节点)
- [Web 面板使用](#web-面板使用)
  - [仪表盘](#仪表盘)
  - [代理策略组](#代理策略组)
  - [规则管理](#规则管理)
  - [连接管理](#连接管理)
  - [日志查看](#日志查看)
- [节点协议支持](#节点协议支持)
- [流量转发模式](#流量转发模式)
- [配置文件详解](#配置文件详解)
- [高级用法](#高级用法)
  - [规则集 (Rule-Set)](#规则集-rule-set)
  - [DNS 配置](#dns-配置)
  - [TUN 模式](#tun-模式)
  - [API 管理](#api-管理)
- [常见问题](#常见问题)

---

## 什么是 FlClash？

FlClash 是一款基于 **Clash Meta (mihomo)** 内核的多平台代理客户端，提供：

- 🎛️ **Web 控制面板** —— 浏览器即可管理，无需安装 GUI
- 🔄 **多协议支持** —— VMess / VLESS / Shadowsocks / Trojan / Hysteria2 / TUIC 等
- 📡 **订阅管理** —— 支持远程订阅 URL，自动更新
- 🧩 **规则系统** —— 灵活的流量分流规则（域名、IP、进程等）
- 🐳 **Docker 原生** —— 轻量部署，资源占用极低

## Docker 部署

### 快速启动

```bash
docker run -d --restart always \
  --name flclash \
  -p 9090:9090 \
  -p 7890:7890 \
  -p 7891:7891 \
  -v /etc/flclash:/root/.config/clash \
  docker.io/frgpmhhh/flclash:latest
```

端口说明：

| 端口 | 用途 |
|------|------|
| `9090` | Web 管理面板 (HTTP) |
| `7890` | HTTP 代理 |
| `7891` | SOCKS5 代理 |
| `7892` | 混合代理 (HTTP + SOCKS) |
| `7893` | TUN 模式（可选） |

### Docker Compose

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  flclash:
    image: docker.io/frgpmhhh/flclash:latest
    container_name: flclash
    restart: always
    ports:
      - "9090:9090"    # Web 管理面板
      - "7890:7890"    # HTTP 代理
      - "7891:7891"    # SOCKS5 代理
      - "7892:7892"    # 混合代理
    volumes:
      - ./config:/root/.config/clash   # 配置文件持久化
      - ./ui:/root/.config/flclash/ui  # 自定义 UI（可选）
    environment:
      - TZ=Asia/Shanghai
```

启动：

```bash
docker compose up -d
```

首次启动后，访问 `http://<服务器IP>:9090` 进入 Web 面板。

## 配置订阅

### 1️⃣ 远程订阅 URL

FlClash 支持从订阅链接自动拉取节点配置：

1. 打开 Web 面板 → `设置` → `订阅`
2. 填入订阅 URL
3. 点击「导入」
4. 点击「更新订阅」

如果订阅需要 UserInfo（如机场），可在设置中填写用户名/密码。

### 2️⃣ 本地配置文件

直接编辑配置文件 `/etc/flclash/config.yaml`（或 volume 挂载的 `config/config.yaml`）：

```yaml
# HTTP(S) 代理端口
port: 7890

# SOCKS5 代理端口
socks-port: 7891

# 混合代理端口
mixed-port: 7892

# 外部控制 (Web 面板)
external-controller: 0.0.0.0:9090
external-ui: /root/.config/flclash/ui

# 允许局域网连接
allow-lan: true
bind-address: "0.0.0.0"
mode: rule
log-level: info
ipv6: true

# 代理节点
proxies:
  # VLESS + TCP + TLS + Vision (XTLS)
  - name: "东京-VLESS"
    type: vless
    server: hy2cn.duckdns.org
    port: 443
    uuid: "你的UUID"
    network: tcp
    tls: true
    udp: true
    flow: xtls-rprx-vision
    servername: hy2cn.duckdns.org

  # Hysteria2
  - name: "东京-HY2"
    type: hysteria2
    server: hy2cn.duckdns.org
    port: 8443
    password: "你的密码"
    sni: bing.com
    skip-cert-verify: false

  # Shadowsocks
  - name: "SS-日本"
    type: ss
    server: example.com
    port: 8388
    cipher: chacha20-ietf-poly1305
    password: "your-password"

  # VMess + WebSocket + TLS
  - name: "VMess-WS"
    type: vmess
    server: example.com
    port: 443
    uuid: "your-uuid"
    alterId: 0
    cipher: auto
    network: ws
    ws-opts:
      path: /websocket
      headers:
        Host: example.com
    tls: true

  # Trojan
  - name: "Trojan"
    type: trojan
    server: example.com
    port: 443
    password: "your-password"
    udp: true
    sni: example.com

  # TUIC v5
  - name: "TUIC"
    type: tuic
    server: example.com
    port: 50000
    token: "your-token"
    udp: true
    ip: 1.2.3.4

# 代理组（策略组）
proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - "♻️ 自动选择"
      - "🎯 直连"
      - "东京-VLESS"
      - "东京-HY2"
      - "SS-日本"
      - "VMess-WS"

  - name: "♻️ 自动选择"
    type: url-test
    url: "http://www.gstatic.com/generate_204"
    interval: 300
    tolerance: 50
    proxies:
      - "东京-VLESS"
      - "东京-HY2"
      - "SS-日本"

  - name: "🎯 直连"
    type: select
    proxies:
      - "DIRECT"

# 规则
rules:
  - "DOMAIN-SUFFIX,google.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,youtube.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,github.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,telegram.org,🚀 节点选择"
  - "DOMAIN-SUFFIX,twitter.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,instagram.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,spotify.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,netflix.com,🚀 节点选择"
  - "DOMAIN-SUFFIX,baidu.com,🎯 直连"
  - "DOMAIN-SUFFIX,cn,🎯 直连"
  - "MATCH,🚀 节点选择"
```

修改后重启容器：

```bash
docker restart flclash
```

### 3️⃣ 手动添加节点

在 Web 面板 → `代理` → 点击「添加节点」，支持粘贴以下格式：

- **Share Link**: `vless://...`、`ss://...`、`trojan://...` 等
- **Clash 配置片段**: Yaml 格式的节点配置

## Web 面板使用

### 仪表盘

`http://<服务器IP>:9090` 首页显示：

- **流量统计** —— 实时上传/下载速度
- **连接数** —— 当前活跃连接
- **内存使用** —— 进程资源占用
- **运行时间** —— 服务持续运行时长

### 代理策略组

> 相当于 Clash 的 `Proxy Group` 管理界面

- **手动选择** —— 点击节点名切换出口
- **自动选择** —— 根据延迟/丢包率自动切换
- **故障转移** —— 节点不可用时自动切换
- **负载均衡** —— 流量分散到多个节点
- **直连** —— 不经过代理

### 规则管理

Web 面板支持：

| 操作 | 说明 |
|------|------|
| 查看规则 | 按优先级列出所有规则 |
| 切换模式 | `全局` / `规则` / `直连` |
| 测试延迟 | 一键测试所有节点延迟 |

### 连接管理

实时查看所有活跃连接：

- 查看每个连接的目标域名/IP
- 查看连接使用的规则和策略组
- 手动关闭特定连接
- 按进程/域名筛选

### 日志查看

`http://<服务器IP>:9090/#/logs`

- `info` — 常规日志
- `warning` — 警告信息
- `error` — 错误信息
- `debug` — 调试信息（最详细）

## 节点协议支持

| 协议 | 支持 | 说明 |
|------|------|------|
| VLESS | ✅ | 支持 XTLS Vision / XTLS RPRX |
| VMess | ✅ | 支持 WebSocket / TCP / gRPC |
| Shadowsocks | ✅ | 支持 AEAD 加密 |
| Trojan | ✅ | 支持 TLS / gRPC |
| Hysteria2 | ✅ | 基于 QUIC，抗丢包 |
| TUIC v5 | ✅ | 基于 QUIC，低延迟 |
| SOCKS5 | ✅ | 支持 UDP |
| HTTP(S) | ✅ | 标准 HTTP 代理 |

## 流量转发模式

FlClash 支持三种代理模式，可在 Web 面板左上角切换：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **规则 (Rule)** | 根据规则集分流 | 🌟 日常使用 |
| **全局 (Global)** | 所有流量走代理 | 测试节点连通性 |
| **直连 (Direct)** | 所有流量不经过代理 | 访问国内网站 |

## 配置文件详解

### 核心配置字段

```yaml
port: 7890                    # HTTP 代理端口
socks-port: 7891              # SOCKS5 端口
mixed-port: 7892              # 混合端口（HTTP + SOCKS5）
allow-lan: true               # 允许局域网连接
bind-address: "0.0.0.0"      # 监听地址
mode: rule                    # 模式: rule / global / direct
log-level: info               # 日志级别: debug / info / warning / error
ipv6: true                    # 启用 IPv6
external-controller: :9090    # 外部控制 API 地址
external-ui: /ui              # Web UI 目录
```

### 代理组类型

| 类型 | 说明 |
|------|------|
| `select` | 手动选择节点 |
| `url-test` | 根据延迟测试自动选择 |
| `fallback` | 按优先级故障切换 |
| `load-balance` | 负载均衡 |

### 规则语法

```
# 域名匹配
DOMAIN,google.com,Proxy
DOMAIN-SUFFIX,youtube.com,Proxy
DOMAIN-KEYWORD,google,Proxy

# IP 匹配
IP-CIDR,10.0.0.0/8,DIRECT
IP-CIDR6,::1/128,DIRECT

# 地理位置
GEOIP,CN,DIRECT

# 最终匹配
MATCH,Proxy
```

## 高级用法

### 规则集 (Rule-Set)

使用远程规则集自动更新分流规则：

```yaml
rule-providers:
  reject:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/reject.txt"
    path: ./ruleset/reject.yaml
    interval: 86400

  proxy:
    type: http
    behavior: domain
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/proxy.txt"
    path: ./ruleset/proxy.yaml
    interval: 86400

  cn:
    type: http
    behavior: ipcidr
    url: "https://cdn.jsdelivr.net/gh/Loyalsoldier/clash-rules@release/cn.txt"
    path: ./ruleset/cn.yaml
    interval: 86400

rules:
  - "RULE-SET,reject,REJECT"
  - "RULE-SET,proxy,🚀 节点选择"
  - "RULE-SET,cn,DIRECT"
  - "MATCH,🚀 节点选择"
```

### DNS 配置

```yaml
dns:
  enable: true
  listen: 0.0.0.0:53
  ipv6: true
  default-nameserver:
    - 223.5.5.5        # 阿里 DNS
    - 119.29.29.29     # 腾讯 DNS
  nameserver:
    - https://doh.alidns.com/dns-query
    - https://doh.pub/dns-query
  fallback:
    - https://dns.google/dns-query
    - https://cloudflare-dns.com/dns-query
  fallback-filter:
    geoip: true
    geoip-code: CN
```

### TUN 模式

TUN 模式可以代理所有系统流量，不需要在应用层设置代理：

```yaml
tun:
  enable: true
  stack: system                   # system / gvisor / mixed
  dns-hijack:
    - any:53
  auto-route: true
  auto-detect-interface: true
```

启动 TUN 模式需要容器添加 `--cap-add NET_ADMIN`：

```bash
docker run -d --restart always \
  --name flclash \
  --cap-add NET_ADMIN \
  --device /dev/net/tun \
  -p 9090:9090 \
  -v /etc/flclash:/root/.config/clash \
  docker.io/frgpmhhh/flclash:latest
```

### API 管理

FlClash 的 Web 面板基于 REST API，可以直接用 curl 管理：

```bash
# 获取节点延迟
curl -X PUT http://localhost:9090/proxies/🚀%20节点选择/delay -d "url=http://www.gstatic.com/generate_204&timeout=5000"

# 切换节点
curl -X PUT http://localhost:9090/proxies/🚀%20节点选择 -d '{"name":"东京-VLESS"}'

# 切换模式
curl -X PATCH http://localhost:9090/configs -d '{"mode":"global"}'

# 查看活跃连接
curl http://localhost:9090/connections

# 关闭所有连接
curl -X DELETE http://localhost:9090/connections
```

## 常见问题

### Q: Web 面板打不开

```bash
# 确认容器在运行
docker ps | grep flclash

# 检查端口映射
docker port flclash

# 检查防火墙
sudo iptables -L INPUT -n | grep 9090
# 或 firewalld
sudo firewall-cmd --list-all
```

### Q: 节点连接超时 / 延迟过高

- 检查节点地址和端口是否可通：`nc -zv <服务器> <端口>`
- 确认订阅链接是否过期
- 尝试切换到其他节点测试
- 检查 `log-level: debug` 查看详细日志

### Q: 容器重启后配置丢失

确认配置文件已持久化挂载：

```yaml
volumes:
  - ./config:/root/.config/clash
```

配置存储在 `/root/.config/clash/config.yaml`。

### Q: 如何更新 FlClash 版本？

```bash
docker pull docker.io/frgpmhhh/flclash:latest
docker compose down
docker compose up -d
```

### Q: 与现有 V2Ray / Xray 节点兼容吗？

兼容。FlClash 支持 VLESS / VMess / Trojan 等协议，只需将节点信息按 Clash 配置格式写入即可。支持 Share Link 直接导入。

### Q: 如何与 VLESS + XTLS Vision 节点配合？

参考上面的配置示例，关键字段：

```yaml
- name: "东京-VLESS"
  type: vless
  server: hy2cn.duckdns.org
  port: 443
  uuid: "你的UUID"
  network: tcp
  tls: true
  udp: true
  flow: xtls-rprx-vision
  servername: hy2cn.duckdns.org
```

### Q: 如何配合 Hysteria2 节点使用？

```yaml
- name: "东京-HY2"
  type: hysteria2
  server: hy2cn.duckdns.org
  port: 8443
  password: "你的密码"
  sni: bing.com
  skip-cert-verify: false
```

### Q: 可以同时运行多个 FlClash 实例吗？

可以，但需要修改端口避免冲突：

```bash
# 实例 1（默认）
docker run -d --name flclash-1 -p 9090:9090 ... frgpmhhh/flclash

# 实例 2（不同端口）
docker run -d --name flclash-2 \
  -p 9091:9090 \
  -p 7893:7890 \
  -v /etc/flclash2:/root/.config/clash \
  frgpmhhh/flclash
```

---

## 参考链接

- [FlClash GitHub](https://github.com/caizhenwei33-cmyk/flclash-docker-tutorial)
- [Clash Meta (mihomo) 文档](https://wiki.metacubex.one/)
- [Clash 规则集 - Loyalsoldier](https://github.com/Loyalsoldier/clash-rules)
- [Clash 配置示例](https://github.com/ACL4SSR/ACL4SSR)

---

> **免责声明**：本教程仅用于技术学习和研究。请遵守当地法律法规，合理使用代理工具。
