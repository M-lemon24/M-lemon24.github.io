---
title: DNS的排查标准顺序
link: DNS排查
catalog: true
date: 2026-11-10 16:00:00
description: DNS网络故障问题的排查
tags:
  - 运维
  - linux
categories:
  - 笔记
draft: false
---
## 一.背景
在一台阿里云香港域ECS（Ubunntu）上部署一个dsh，服务器已将加入tailscale组网，ssh以及ping都正常
## 二.症状
下载时使用任何需要域名解析的命令全部失效:
```bash
$ curl -fsSL -o node.tar.xz https://npmmirror.com/mirrors/node/v24.15.0/node-v24.15.0-linux-x64.tar.xz
curl: (6) Could not resolve host: npmmirror.com

$ curl -fsSL -o node.tar.xz https://nodejs.org/dist/...
curl: (6) Could not resolve host: nodejs.org

$ getent hosts baidu.com
（无任何输出）
```
这里的curl（6）时curl的错误码，含义是“域名解释失败”。


同时之前我想在服务器上面部署一个openAI也出现这种情况
```bash
$ curl -s --max-time 10 https://api.openai.com/v1/models
000
```
这里的000表示“HTTP响应都没有拿到”，当时只想着香港IP是不在官方GPT的注册范围内，其实也是因为DNS故障早城的——解析失败，根本就不能访问。
## 三.排查时“网络问题”还是“DNS问题”
```bash
$ ping -c 2 8.8.8.8
回显
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=1.14 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=1.13 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss

$ getent hosts baidu.com
(无输出)
```
这里的ping 8.8.8.8 结果显示1.14ms 说明IP层网络完全正常

而getent hosts baidu.com 结果显示 没有输出 说明域名解析失败

IP通+域名不同=DNS问题

在这里说一下gentent

getent=get entries（获取条目），它用来查询Linux系统的各种信息数据库。

hosts 是其中的子命令，作用是：解析主机名到IP，和DNS查询效果一样
```bash
$ getent hosts baidu.com
#正常情况下
104.20.28.246   nodejs.org
```
它与nslookup/dig的区别
| 命令 | 走哪条路 | 代表什么 |
| :--- | :--- | :--- |
|`nslookup baidu.com` | 直接问某个DNS服务器 | 只看「那个 DNS 服务器能不能解析」 |
|`dig @8.8.8.8.8 baidu.com` | 直接问8.8.8.8.8 | 只测试指定的那一台 |
|`gentent hosts baidu.com` | 走系统的完整解析链路 | 程序实际会遇到的结果 |

## 四.逐层排查
DNS解析是一条链，要逐层验证：
```
应用程序
   ↓
NSS（/etc/nsswitch.conf）
   ↓
/etc/resolv.conf   ←  指向谁
   ↓
本地解析服务（systemd-resolved）
   ↓
上游 DNS 服务器     ←  真的通吗？
```
### 4.1 看/etc/resolv.conf指向哪里
```bash
$ cat /etc/resolv.conf
nameserver 127.0.0.53
options edns0 trust-ad
search tail2d0202e9.ts.net
```
这个回显有两个关键信息
1.nameserver 127.0.0.53 -这里不是真实的DNS，而是systemd-resolved在本机开的Stub监听端口。他只负责转发。

2.search tail2d0202e9.ts.net-这是Tailscale写入的MagicDNS搜索域。

也就是说，真正干活的不是这个问渐渐，而是systemd-resolved服务。

### 4.2 确认systemd-resolved是否在运行
```bash
$ systemctl status systemd-resolved
● systemd-resolved.service - Network Name Resolution
     Loaded: loaded (/lib/systemd/system/systemd-resolved.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-25 06:36:21 CST; 1 week 2 days ago
   Main PID: 49117 (systemd-resolve)
```
这里的running表示服务还存在着，这一层就排除掉了

顺便提议一下，底部的（END）是less分页器，按q退出。，要是想避免这个END就加上--no -pager，或管道给cat，或直接systemctl is-active <服务名>。

、
### 4.3 看它实际配了哪些上游的DNS
```bash
$ resolvectl status
Global
        Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
   resolv.conf mode: stub

Link 2 (eth0)
     Current Scopes: DNS
        Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
   Current DNS Server: 100.100.2.138
        DNS Servers: 100.100.2.136 100.100.2.138

Link 3 (tailscale0)
     Current Scopes: DNS
        Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
        DNS Servers: 100.100.100.100 fd7a:115c:a1e0::53
        DNS Domain: tail2d02e9.ts.net ~100.in-addr.arpa ...
```
解释一下各个字段的含义
| 字段 | 含义 |
| :--- | :--- |
| `resolv.conf mode: stub` | `/etc/resolv.conf` 用 stub 模式，依赖 systemd-resolved |
| `+DefaultRoute` | **加号** = 默认 DNS 出口，普通域名都走它 |
| `-DefaultRoute` | **减号** = 非默认，只有匹配 `DNS Domain` 的域名才走它 |
| `DNS Servers:` | 该网卡的上游 DNS 服务器 |

实际意思就是
eth0：所有普通域名都问100.100.2.136/100.100.2.138

tailscale0：只有*.ts.net和100.x反解去问100.100.100.100

这个配置本身是没有任何问题的，而且tailscale那条显示-DefaultRoute，说明它没有抢占默认DNS

### 4.4 这个上游真的通了吗
```bash
$ resolvectl query baidu.com
baidu.com: resolve call failed: Query timed out

$ ping -c 2 100.100.2.136
PING 100.100.2.136 (100.100.2.136) 56(84) bytes of data.

--- 100.100.2.136 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1019ms
```
这里的100.100.2.136  100%丢包，根本不可达。

## 五.原因
刚才的配置文件里面写的100.100.2.136 / 100.100.2.138 是阿里云国内地域的内网 DNS 地址。
香港等国际地区通常使用不同的DNS服务，但是这台实例的DHCP沿用了国内的模板，下发了一个在港区不同的地址。
完整的故障链
```
程序查域名
   ↓
/etc/resolv.conf → 127.0.0.53
   ↓
systemd-resolved（运行中，应答正常）
   ↓
上游 DNS 100.100.2.136   ❌ 100% 丢包，永久超时
   ↓
resolve call failed: Query timed out
```
其实从刚才确认systemd-resolved是否在运行里面回显出现的也能看出来
```
systemd-resolved[49117]: Using degraded feature set UDP instead of TCP
```
这个的意思就是TCP 连上游失败,退化成 UDP。这正是上游不可达的典型表现。

## 六、解决方案

### 6.1 临时验证(10 秒见效)

```bash
sudo resolvectl dns eth0 8.8.8.8 1.1.1.1
resolvectl query baidu.com
```

result:

```
baidu.com: 110.242.74.102   -- link: eth0
           111.63.65.103
           111.63.65.247
           124.237.177.164
-- Information acquired via protocol DNS in 3.6ms.
```

**3.6ms,瞬间恢复。** 方向确认无误。

但这是**运行时配置,重启或网络变化后会失效**(DHCP 会把坏地址写回来)。

### 6.2 永久修复

**方案 A:静态 `/etc/resolv.conf`**

```bash
# 1. 停用 systemd-resolved，防止它改回去
sudo systemctl disable --now systemd-resolved

# 2. 写入静态 DNS
sudo rm -f /etc/resolv.conf
sudo tee /etc/resolv.conf > /dev/null <<'EOF'
nameserver 8.8.8.8
nameserver 1.1.1.1
nameserver 223.5.5.5
EOF

# 3. 锁死，防止任何程序覆盖
sudo chattr +i /etc/resolv.conf

# 4. 验证
getent hosts nodejs.org
```

> `chattr +i` 把文件设为 **immutable**,连 root 都不能改,必须 `chattr -i` 解锁。这是防止 DNS 配置被 DHCP / VPN / 云助手反复覆盖的关键一步。

**代价**:Tailscale MagicDNS(用机器名访问)失效。但用 `100.x.x.x` IP 访问不受影响。

**方案 B:修改 netplan(更规范,保留 systemd-resolved)**

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      dhcp4-overrides:
        use-dns: false          # 忽略 DHCP 下发的 DNS
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

```bash
sudo netplan apply
```

> **注意**:netplan 缩进错一格就会**完全断网**,而云服务器只能通过 VNC 恢复。对无本地控制台的机器,方案 A 更安全。

---

## 七、可用的公共 DNS 清单

在香港/海外服务器上实测可用:

| DNS | 提供方 | 特点 |
|---|---|---|
| `8.8.8.8` / `8.8.4.4` | Google | 实测 1.14ms,全球任播 |
| `1.1.1.1` / `1.0.0.1` | Cloudflare | 隐私性好 |
| `9.9.9.9` | Quad9 | 拦截恶意域名 |
| `223.5.5.5` | 阿里公共 DNS | 国内快 |
| `119.29.29.29` | 腾讯 DNSPod | 国内快 |

**一键批量测试哪个可用:**

```bash
for d in 8.8.8.8 1.1.1.1 223.5.5.5 119.29.29.29 9.9.9.9; do
  printf "%-16s " "$d"
  timeout 3 nslookup baidu.com "$d" >/dev/null 2>&1 && echo "✅ 可用" || echo "❌ 不可用"
done
```

---

## 八、排查方法论

把这次的经验整理成一个决策树,下次照着走:

```
域名解析失败
    │
    ├─ ① ping 8.8.8.8
    │      失败 → 网络层问题，查安全组/路由/网卡
    │      成功 → 继续
    │
    ├─ ② cat /etc/resolv.conf
    │      空的 → 补 nameserver
    │      是 127.0.0.53 → 依赖 systemd-resolved，继续
    │
    ├─ ③ systemctl is-active systemd-resolved
    │      inactive → 启动它，或改用静态 resolv.conf
    │      active → 继续
    │
    ├─ ④ resolvectl status
    │      看 DNS Servers 是否为空、是否可达
    │
    ├─ ⑤ resolvectl query 目标域名
    │      超时 → 上游 DNS 不通，换 DNS
    │
    └─ ⑥ ping <上游DNS IP>
           丢包 → 确认根因，换掉它
```

### `getent hosts` 为什么比 `nslookup` 更适合做判断

| 命令 | 走哪条路 | 代表什么 |
| :--- | :--- | :--- |
| `nslookup baidu.com` | 直接问某个 DNS 服务器 | 只看「那个 DNS 服务器能不能解析」 |
| `dig @8.8.8.8 baidu.com` | 直接问 8.8.8.8 | 只测试指定的那一台 |
| `getent hosts baidu.com` | 走系统的完整解析链路 | ✅ 程序实际会遇到的结果 |

`getent` 会依次查 `/etc/nsswitch.conf` → `/etc/hosts` → systemd-resolved → DNS,**和 curl、ping、npm 走的完全是同一条路**。

**所以:**

- `nslookup @8.8.8.8` 成功 + `getent hosts` 失败 = **DNS 服务器没问题,是系统配置没用上它**
- 只跑 `nslookup` 会误判,以为 DNS 是好的

---

## 九、同期踩到的其他坑(附赠)

这次部署过程中还遇到几个值得记录的问题。

### 9.1 Node.js 版本过低导致"静默失败"

```bash
$ dsh --version
（无任何输出，退出码 0）
```

安装看似成功(`added 616 packages`),但程序**完全没有任何输出,也不工作**。

根因:部分依赖要求 `node >= 22.19.0`,而系统装的是 **v22.14.0**。升级到 **Node 24** 后立刻正常:

```bash
$ dsh --version
0.2.0-rc.2
```

> **教训:安装 Node.js 应用时,一定先看 `npm install` 输出里的 `EBADENGINE` 警告。**
> 它不是"提醒",而是"这个包在你这里可能根本不工作"。

### 9.2 磁盘写满导致安装中途失败

```bash
$ df -h /
/dev/mapper/ubuntu--vg-ubuntu--lv   17G   15G  93M  100% /
```

npm 安装到一半磁盘写满,产生的文件不完整,但**没有报错**。

**清理思路(按收益排序):**

```bash
sudo journalctl --vacuum-size=50M   # 系统日志，通常能省几百 M
sudo apt-get clean                  # 包缓存
npm cache clean --force             # npm 缓存
# 删除 snap 旧版本
sudo snap list --all | awk '$6=="disabled"{print $1, $3}'
```

**还有一个常被忽略的地方:LVM 未分配空间。**

```bash
$ lsblk
vda                        35G   ← 物理盘 35G
├─vda2                      2G
└─vda3                     33G    ← 分区 33G
  └─ubuntu--vg-ubuntu--lv 16.5G   ← 逻辑卷只用了 16.5G！
```

**分区 33G,逻辑卷只用了 16.5G —— 剩下 16G 一直闲置着。** 一条命令就能在线扩容:

```bash
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

**17G → 33G,无需重启、不影响服务。**

> **教训:磁盘告急时,先看 `lsblk`,可能有免费空间在睡觉。**

### 9.3 `pkill -f` 把自己杀了

```bash
pkill -9 -f "bin/dsh"
```

这条命令匹配的是**完整命令行**。而执行它的 shell 进程,命令行里恰好包含 `bin/dsh` 这个字符串 —— **于是把自己也杀了**,脚本执行中断。

**正确做法**:按端口找 PID,精确定位:

```bash
for p in $(ss -tlnp | grep ':3080 ' | grep -oP 'pid=\K[0-9]+' | sort -u); do
    kill -9 "$p"
done
```

---

## 十、总结

| 问题 | 根因 | 解决 |
|---|---|---|
| 域名全解析失败 | 云厂商下发的内网 DNS 在港区不可达 | 换公共 DNS + `chattr +i` 锁定 |
| `dsh --version` 无输出 | Node 22.14 < 要求的 22.19 | 升级 Node 24 |
| 安装过程无报错但失败 | 磁盘 100% 满 | 清理 + LVM 扩容 |
| 脚本执行到一半中断 | `pkill -f` 匹配到自身 | 按端口精确杀进程 |

**三条最值得记住的原则:**

1. **IP 通 + 域名不通 = DNS 问题**,一条 `ping 8.8.8.8` + `getent hosts` 就能切开
2. **`getent hosts` 比 `nslookup` 更接近真实**,它走的是应用程序实际用的解析链路
3. **云服务器的 DNS 配置会被反复覆盖**,用 `chattr +i` 锁死才可靠

---

## 参考

- [systemd-resolved 官方文档](https://www.freedesktop.org/software/systemd/man/systemd-resolved.service.html)
- [resolv.conf(5) man page](https://man7.org/linux/man-pages/man5/resolv.conf.5.html)
- [nsswitch.conf(5)](https://man7.org/linux/man-pages/man5/nsswitch.conf.5.html)
