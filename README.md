# iStoreOS / OpenWrt + Lancache DNS 劫持配置指南

[English](#english) | 中文

在 iStoreOS（OpenWrt）主路由上使用 `cache-domains` 插件，将游戏 CDN 域名 DNS 劫持到局域网内的 Lancache 缓存服务器，实现游戏下载加速。

**实测环境**：iStoreOS 24.10.6 + OpenClash + UU 加速器，完美共存，互不影响。

---

## 目录

- [这是什么](#这是什么)
- [适用场景](#适用场景)
- [环境与原理](#环境与原理)
- [前置条件](#前置条件)
- [安装步骤](#安装步骤)
  - [1. 安装插件](#1-安装插件)
  - [2. 创建配置文件](#2-创建配置文件)
  - [3. 生成 DNS 规则](#3-生成-dns-规则)
  - [4. 持久化配置（关键）](#4-持久化配置关键)
  - [5. 创建开机自启脚本](#5-创建开机自启脚本)
  - [6. 验证](#6-验证)
- [更新域名列表](#更新域名列表)
- [卸载与恢复默认 DNS](#卸载与恢复默认-dns)
- [实时查看缓存日志](#实时查看缓存日志)
- [常见问题](#常见问题)
- [踩坑记录](#踩坑记录)

---

## 这是什么

Lancache 是一个自建的游戏内容缓存服务器，可以缓存 Steam、Epic、暴雪、Xbox、PlayStation 等平台的游戏下载内容。局域网内多台设备下载同一游戏时，只有第一次走外网，之后全部从本地 NAS 提供，速度拉满。

但 Lancache 的前提是：**游戏客户端的 DNS 请求必须指向 Lancache 服务器**。

本指南解决的问题是：如何在 **iStoreOS / OpenWrt 主路由**层面实现 DNS 劫持，而不需要修改每台设备的 DNS 设置，同时不影响 OpenClash、UU 加速器等已有插件。

## 适用场景

- 你有一台 NAS（群晖、飞牛、威联通等）运行 Lancache Docker
- 你的路由器运行 iStoreOS 或 OpenWrt
- 路由器上可能还运行着 OpenClash、UU 加速器等插件
- 你想让局域网内所有设备自动走 Lancache 缓存，而不是手动改 DNS

## 环境与原理

```
Internet
    │
    ▼
┌─────────────────┐
│  iStoreOS 路由   │  ← OpenClash / UU 加速器
│  (192.168.x.1)  │
└──────┬──────────┘
       │
       ├── NAS：Lancache Docker（192.168.x.100）
       ├── PC / 游戏机
       └── 其他设备
```

**工作原理**：

1. `cache-domains` 插件从 [uklans/cache-domains](https://github.com/uklans/cache-domains) 拉取最新的游戏 CDN 域名列表
2. 结合你的配置，生成 dnsmasq 规则，将这些域名指向 Lancache 服务器 IP
3. 游戏客户端请求域名时，dnsmasq 返回 NAS 内网 IP → 流量走 Lancache 缓存
4. 其他域名（普通网站、代理流量）完全不受影响

**兼容性**：

| 插件 | 是否受影响 | 原因 |
|------|-----------|------|
| OpenClash | ❌ 不影响 | dnsmasq 层劫持早于 Clash DNS 处理 |
| UU 加速器 | ❌ 不影响 | 只劫持游戏 CDN 域名，不碰加速器流量 |
| 其他 DNS 插件 | 视情况 | 如果有 SmartDNS / AdGuardHome 需要注意端口冲突 |

## 前置条件

- iStoreOS 或 OpenWrt 路由器（已联网）
- 局域网内有 Lancache 服务器（已部署并运行）
- 路由器可以 SSH 登录
- 路由器可以访问 GitHub（用于拉取域名列表，不能访问见[常见问题](#q-github-连不上怎么办)）

---

## 安装步骤

### 1. 安装插件

```bash
opkg update
opkg install cache-domains-mbedtls
```

> 你也可以安装 `cache-domains-openssl` 或 `cache-domains-wolfssl`，功能完全相同，只是底层 SSL 库不同。mbedtls 版本在 iStoreOS 上最常见。

安装后的文件结构：

| 文件 | 路径 | 用途 |
|------|------|------|
| 主脚本 | `/usr/bin/cache-domains` | 拉取域名列表、生成 dnsmasq 规则 |
| 热插拔触发器 | `/etc/hotplug.d/iface/30-cache-domains` | WAN 上线自动触发 |
| 测试脚本 | `/usr/share/cache-domains/test.sh` | 验证配置 |

### 2. 创建配置文件

创建 `/etc/cache-domains.json`，将 `YOUR_NAS_IP` 替换为你的 Lancache 服务器内网 IP：

```bash
cat > /etc/cache-domains.json << 'EOF'
{
    "ips": {
        "nas": "YOUR_NAS_IP"
    },
    "cache_domains": {
        "default": "nas"
    }
}
EOF
```

> `default: "nas"` 表示所有游戏 CDN 域名都解析到 NAS。如果你想精细控制（比如多台缓存服务器），可以参考下方的进阶配置。

<details>
<summary>进阶配置：多台缓存服务器 / 按平台分流</summary>

```json
{
    "ips": {
        "nas1": "192.168.1.100",
        "nas2": "192.168.1.101"
    },
    "cache_domains": {
        "default": "nas1",
        "steam": "nas1",
        "epicgames": "nas1",
        "blizzard": "nas2",
        "origin": "nas2",
        "riot": "nas2",
        "sony": "nas1",
        "nintendo": "nas1",
        "xboxlive": "nas1"
    }
}
```

可用的分类名：`steam`、`epicgames`、`blizzard`、`origin`、`riot`、`ubisoft`、`sony`、`nintendo`、`xboxlive`、`wsus` 等，来自 [uklans/cache-domains](https://github.com/uklans/cache-domains) 的 `.txt` 文件名。

</details>

### 3. 生成 DNS 规则

```bash
/usr/bin/cache-domains configure
```

这一步会：
1. 从 GitHub 下载最新域名列表
2. 结合你的配置生成 dnsmasq `.conf` 文件
3. 解压到 `/var/cache-domains/uklans-cache-domains-*/`

**验证是否生成成功**：

```bash
ls /var/cache-domains/uklans-cache-domains-*/scripts/output/dnsmasq/
```

如果看到 `steam.conf`、`epicgames.conf`、`blizzard.conf` 等文件，说明生成成功。

> ⚠️ 运行时可能看到 `udhcpc: started ... no lease, failing`——这是无关输出，不影响结果。

### 4. 持久化配置（关键）

这一步是整个配置中**最容易踩坑**的地方，请仔细阅读。

#### 为什么不能直接 `uci add_list confdir`？

iStoreOS 的 dnsmasq 使用带 hash 后缀的临时目录（如 `/tmp/dnsmasq.cfg01411c.d`），且 OpenClash 会在此目录注入大量规则。直接通过 UCI 添加 `confdir` 会破坏 OpenClash 的规则链，导致 **dnsmasq 崩溃**（`nslookup: Connection refused`）。

#### 正确做法

创建一个持久目录，存放生成的规则文件，然后通过自启脚本在开机时复制到 dnsmasq 的实际加载目录：

```bash
# 创建持久目录
mkdir -p /etc/dnsmasq.d

# 复制生成的规则到持久目录
cp /var/cache-domains/uklans-cache-domains-*/scripts/output/dnsmasq/*.conf /etc/dnsmasq.d/

# 立即生效（复制到 dnsmasq 实际加载的临时目录）
cp /etc/dnsmasq.d/*.conf /tmp/dnsmasq.cfg*.d/

# 重启 dnsmasq
/etc/init.d/dnsmasq restart
```

### 5. 创建开机自启脚本

为了在重启后自动加载规则（且不破坏 OpenClash），创建一个开机自启服务：

```bash
cat > /etc/init.d/lancache-dns << 'EOF'
#!/bin/sh /etc/rc.common
START=99
STOP=10

start() {
    sleep 5
    mkdir -p /etc/dnsmasq.d
    DNSMASQ_CONF_DIR=$(ls -d /tmp/dnsmasq.cfg*.d 2>/dev/null | head -n 1)
    if [ -n "$DNSMASQ_CONF_DIR" ] && [ -n "$(ls /etc/dnsmasq.d/*.conf 2>/dev/null)" ]; then
        echo "Lancache: Copying config to $DNSMASQ_CONF_DIR"
        cp /etc/dnsmasq.d/*.conf "$DNSMASQ_CONF_DIR/"
        /etc/init.d/dnsmasq reload
    else
        echo "Lancache: No .conf files found in /etc/dnsmasq.d/"
    fi
}

stop() {
    DNSMASQ_CONF_DIR=$(ls -d /tmp/dnsmasq.cfg*.d 2>/dev/null | head -n 1)
    if [ -n "$DNSMASQ_CONF_DIR" ]; then
        grep -rl "YOUR_NAS_IP" "$DNSMASQ_CONF_DIR" 2>/dev/null | xargs rm -f
        /etc/init.d/dnsmasq reload
    fi
}
EOF
```

> ⚠️ 将脚本中的 `YOUR_NAS_IP` 替换为你的实际 NAS IP。

赋予权限并开启自启：

```bash
chmod +x /etc/init.d/lancache-dns
/etc/init.d/lancache-dns enable
/etc/init.d/lancache-dns start
```

验证自启链接：

```bash
ls -la /etc/rc.d/S99lancache-dns
# 预期输出：lrwxrwxrwx  ...  /etc/rc.d/S99lancache-dns -> ../init.d/lancache-dns
```

### 6. 验证

```bash
# 测试游戏域名 → 应返回你的 NAS IP
nslookup xvcf2.xboxlive.com
nslookup steamcontent.com
nslookup download.epicgames.com
nslookup cdn.blizzard.com

# 测试普通网站 → 应返回公网 IP
nslookup baidu.com
nslookup google.com
```

**重启路由器后再验证一次**，确认开机自启正常。

---

## 更新域名列表

游戏平台可能新增 CDN 域名，建议定期更新：

```bash
rm -f /var/cache-domains/lancache.conf
/usr/bin/cache-domains configure
cp /var/cache-domains/uklans-cache-domains-*/scripts/output/dnsmasq/*.conf /etc/dnsmasq.d/
/etc/init.d/lancache-dns start
```

---

## 卸载与恢复默认 DNS

### 快速一键卸载

```bash
/etc/init.d/lancache-dns stop
/etc/init.d/lancache-dns disable
rm /etc/init.d/lancache-dns
rm -rf /etc/dnsmasq.d/
rm -rf /var/cache-domains/
rm -f /etc/cache-domains.json
opkg remove cache-domains-mbedtls
/etc/init.d/dnsmasq restart
```

### 验证已恢复

```bash
nslookup steamcontent.com
# 应返回公网 IP，而非你的 NAS IP
```

### 卸载后检查清单

| 检查项 | 命令 | 通过条件 |
|--------|------|---------|
| 无残留规则 | `grep -r "YOUR_NAS_IP" /tmp/dnsmasq.cfg*/` | 无输出 |
| 域名返回公网 IP | `nslookup steamcontent.com` | 非内网 IP |
| 插件已移除 | `opkg list-installed \| grep cache-domains` | 无输出 |

---

## 实时查看缓存日志

配置完成后，可以通过 Lancache 的日志确认缓存是否生效。

### 查看 Docker 日志

```bash
docker logs -f lancache
docker logs --tail 100 -f lancache
```

### 查看日志文件

```bash
tail -f /path/to/lancache/logs/access.log
tail -f /path/to/lancache/logs/error.log
```

### 日志状态说明

| 状态 | 含义 |
|------|------|
| `MISS` | 本地没有缓存，正在从互联网下载并写入缓存 |
| `HIT` | 已命中本地缓存，直接从 NAS 提供数据 |
| `BYPASS` | 请求未缓存，直接透传 |

### 只查看缓存命中

```bash
tail -f /path/to/lancache/logs/access.log | grep HIT
```

### 日志示例

```
[steam] 192.168.1.216 ... "MISS"    # 首次下载，走外网
[steam] 192.168.1.216 ... "HIT"     # 再次下载，走本地缓存
```

---

## 常见问题

### Q: GitHub 连不上怎么办？

`cache-domains configure` 需要从 GitHub 拉取域名列表。如果网络不通：

1. 手动下载 [uklans/cache-domains](https://github.com/uklans/cache-domains) 的 tarball
2. 上传到路由器的 `/var/cache-domains/` 目录并解压
3. 后续步骤照常执行

### Q: 重启后规则失效了？

检查自启脚本是否正确创建：

```bash
ls -la /etc/rc.d/S99lancache-dns
cat /etc/init.d/lancache-dns
```

### Q: OpenClash 配置出问题了？

本方案**不会修改 OpenClash 的任何配置**。如果出现问题，检查是否误执行了 `uci add_list dhcp.@dnsmasq[0].confdir`——这是踩坑操作，详见[踩坑记录](#踩坑记录)。

### Q: 怎么知道 Lancache 是否真的在工作？

1. 在路由器上 `nslookup steamcontent.com`，确认返回 NAS IP
2. 在 NAS 上查看 Lancache 日志，看是否有 `HIT` 记录
3. 下载一个游戏，第二次下载同一内容时速度应该明显更快

### Q: 可以只劫持部分平台吗？

可以，在 `cache_domains` 中只配置你需要的平台：

```json
{
    "cache_domains": {
        "steam": "nas",
        "epicgames": "nas"
    }
}
```

不配置 `default` 的话，其他平台不会被劫持。

---

## 踩坑记录

### 坑 1：`uci add_list confdir` 导致 dnsmasq 崩溃

**现象**：执行 `uci add_list dhcp.@dnsmasq[0].confdir="/etc/dnsmasq.d"` 后，所有 DNS 查询返回 `Connection refused`。

**原因**：OpenClash 在 dnsmasq 的临时目录注入了大量规则（如 `server=/#/127.0.0.1#xxxx`）。通过 UCI 添加新的 `confdir` 会改变 dnsmasq 的配置加载顺序，导致 OpenClash 注入的规则链断裂，dnsmasq 无法启动。

**解决**：不要用 UCI 修改 `confdir`，改用开机脚本在 dnsmasq 启动后手动复制 `.conf` 文件到临时目录。

### 坑 2：`/usr/bin/cache-domains configure` 安装失败

**现象**：运行 `configure` 后 `.conf` 文件没有出现在 dnsmasq 的加载目录中，`nslookup` 返回空。

**原因**：脚本内部的 `dnsmasq_conf()` 函数在尝试操作 UCI 和重启 dnsmasq 时，被 `udhcpc` 的输出干扰，导致后续步骤没有执行。

**解决**：手动完成最后两步——复制 `.conf` 文件 + 重启 dnsmasq。

### 坑 3：dnsmasq 的 conf-dir 带随机 hash

**现象**：dnsmasq 实际加载的目录是 `/tmp/dnsmasq.cfg01411c.d`，而不是你预期的 `/tmp/dnsmasq.d/`。

**原因**：OpenWrt 的 UCI 系统会为每个配置段生成一个 hash 后缀，不同设备 / 不同版本的 hash 值不同。

**解决**：自启脚本中使用 `ls -d /tmp/dnsmasq.cfg*.d` 来自动匹配，而不是硬编码路径。

---

## 文件路径速查

| 用途 | 路径 | 创建方式 |
|------|------|---------|
| 插件配置 | `/etc/cache-domains.json` | 手动创建 |
| 持久化规则 | `/etc/dnsmasq.d/*.conf` | 从生成目录复制 |
| 自启脚本 | `/etc/init.d/lancache-dns` | 手动创建 |
| 自启链接 | `/etc/rc.d/S99lancache-dns` | `enable` 命令自动生成 |
| 临时规则 | `/tmp/dnsmasq.cfg*.d/*.conf` | 由自启脚本自动复制 |

---

## 英文简介 / English

### What is this?

A guide for configuring DNS-based game content caching on iStoreOS / OpenWrt routers using the `cache-domains` package and a LAN Lancache server. Compatible with OpenClash and other proxy/acceleration plugins.

### TL;DR

```bash
opkg update && opkg install cache-domains-mbedtls
# Create /etc/cache-domains.json with your NAS IP
/usr/bin/cache-domains configure
# Copy generated .conf files to /etc/dnsmasq.d/
# Create /etc/init.d/lancache-dns (see guide above)
# Reboot and verify
```

### Tested on

- iStoreOS 24.10.6 (x86_64) + OpenClash + UU Accelerator
- Lancache running in Docker on a NAS

### Key insight

**Do NOT use `uci add_list dhcp.@dnsmasq[0].confdir`** to add a custom conf directory. It will break OpenClash's injected rules and crash dnsmasq. Instead, use an init script (START=99) that copies `.conf` files into dnsmasq's dynamically-named temp directory after dnsmasq starts.

---

## License

MIT

## 致谢

- [uklans/cache-domains](https://github.com/uklans/cache-domains) — 游戏 CDN 域名列表
- [Lancache.net](https://lancache.net) — Lancache 项目
- [OpenWrt](https://openwrt.org) — 路由器固件
- [iStoreOS](https://www.istoreos.com) — OpenWrt 商业发行版
