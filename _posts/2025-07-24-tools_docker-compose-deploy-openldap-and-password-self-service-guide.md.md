---
title: Docker-Compose 部署 OpenLDAP 与密码自助找回服务教程
date: 2025-07-24 17:25:00 +0800
categories: [工具]
tags: [OpenLDAP,ldap]
pin: false
---

## 概述

本教程将指导您使用 Docker-Compose 部署 OpenLDAP 和 LTB Self-Service Password (SSP)，以实现 LDAP 密码的自助找回功能。我们还将添加详细注释，帮助您理解配置文件中的每一项设置。

---

## 前提条件

- 已安装 Docker 和 Docker-Compose。
- 基础的 Linux 操作系统使用经验。

---

## 步骤 1：创建 `docker-compose.yml` 文件

在目标目录下创建 `docker-compose.yml` 文件，粘贴以下内容：

```yaml
docker-compose.yml
services:
  openldap:
    container_name: openldap
    hostname: openldap.aoco.tech
    restart: always
    image: osixia/openldap:latest
    ports:
      - "389:389" # 暴露 LDAP 服务的 389 端口
    environment:
      - LDAP_ORGANISATION="aoco.tech" # 设置 LDAP 组织名称
      - LDAP_DOMAIN=aoco.tech         # 设置 LDAP 根域名
      - LDAP_ADMIN_PASSWORD=abc2024XYZ # 设置 LDAP 管理员密码
      - LDAP_TLS=false # 是否启用 TLS（false 表示不启用，便于测试）
    volumes:
      - /opt/ldap/local:/usr/local/ldap # 本地挂载目录，便于存储自定义内容
      - /opt/ldap/data:/var/lib/ldap # 存储 LDAP 数据的目录
      - /opt/ldap/slapd.d:/etc/ldap/slapd.d # 存储 OpenLDAP 配置文件的目录
      - /etc/certs/ssl/aoco.tech:/container/service/slapd/assets/certs # SSL 证书挂载目录
    command: [--copy-service,  --loglevel, warning] # 设置启动命令，减少日志冗余

  phpldapadmin:
    container_name: phpldapadmin
    hostname: phpldapadmin.aoco.tech
    restart: always
    image: osixia/phpldapadmin:latest
    ports:
      - 8080:80 # 暴露 phpLDAPadmin 的 HTTP 服务
    environment:
      - PHPLDAPADMIN_HTTPS=false # 禁用 HTTPS（便于测试）
      - PHPLDAPADMIN_LDAP_HOSTS=openldap # 指定 OpenLDAP 服务主机名
    links:
      - openldap # 链接到 OpenLDAP 容器
    depends_on:
      - openldap # 确保 OpenLDAP 容器先启动

  ssp-app:
    image: ltbproject/self-service-password
    hostname: ssp.aoco.tech
    container_name: ssp-app
    restart: always
    volumes:
      - ./custom/ssp.conf.php:/var/www/conf/config.inc.local.php:ro # 挂载自定义配置文件
      - ./custom/images:/var/www/htdocs/images/custom:ro # 挂载自定义图片目录
    ports:
      - 80:80 # 暴露 SSP 服务的 HTTP 服务
```

---

## 步骤 2：创建 SSP 的自定义 PHP 配置文件

在同一目录下创建 `custom/ssp.conf.php` 文件，粘贴以下内容，并根据需求调整：

```php
<?php
// Override config.inc.php parameters below

// LDAP 相关配置
$ldap_url = "ldap://openldap"; // OpenLDAP 容器的地址
$ldap_binddn = "cn=admin,dc=algs,dc=tech"; // LDAP 管理员账号
$ldap_bindpw = "abc2025XYZ"; // LDAP 管理员密码
$ldap_base = "dc=algs,dc=tech"; // LDAP 基准 DN

// 禁用短信功能
$use_sms = false; // 禁用通过短信修改密码的功能

// 密码自助服务选项
$use_questions = false; // 不使用密保问题
$who_change_password = "user"; // 允许用户修改自己的密码
$mail_attribute = "mail"; // 使用邮箱验证用户身份
$audit_log_file = '/var/log/ssp_audit.log'; // 定义审计日志存储路径

// 调试选项
$debug = false; // 禁用调试模式

// 密码修改成功提示
$keyphrase = "scsdsicicsiccsd"; // 密码加密的密钥短语
$messages['passwordchangedextramessage'] = "您的密码已成功更改！"; // 用户密码更改成功的提示信息
$messages['changehelpextramessage'] = "如果遇到问题，请联系技术支持：support@aoco.tech"; // 修改失败时的帮助信息
$default_action = "change"; // 默认操作为修改密码

// 自定义界面配置
$background_image = "images/unsplash-sky.jpeg"; // 自定义背景路径
// $logo = "images/custom/logo.png"; // 自定义 Logo 路径（如果需要）

// 邮箱配置
$reset_url = "http://ssp.aoco.tech"; // 密码重置服务的 URL
$mail_address_use_ldap = true; // 从 LDAP 中获取用户邮箱
$use_tokens = true; // 启用令牌功能
$token_lifetime = "3600"; // 令牌有效时间（单位：秒）
$mail_smtp_secure = "ssl"; // 使用 SSL 加密
$mail_from = "noreply@aoco.tech"; // 发件人邮箱
$mail_from_name = "自助密码重置平台"; // 发件人名称
$mail_signature = ""; // 邮件签名
$mail_sendmailpath = '/usr/sbin/sendmail';
$mail_protocol = 'smtp';
$mail_smtp_debug = false; // 禁用 SMTP 调试
$mail_debug_format = 'html'; // 使用 HTML 格式调试信息
$mail_smtp_host = 'smtphz.qiye.163.com'; // SMTP 服务器地址
$mail_smtp_auth = true; // 启用 SMTP 身份验证
$mail_smtp_user = 'noreply@aoco.tech'; // 发件邮箱账号
$mail_smtp_pass = 'password'; // 发件邮箱密码
$mail_smtp_port = 465; // SMTP 端口（465 表示 SSL）
$mail_smtp_timeout = 5; // 超时时间（单位：秒）
$mail_smtp_keepalive = false; // 禁用 SMTP 长连接
$mail_smtp_autotls = true; // 自动启用 TLS
$mail_smtp_options = array(); // 额外的 SMTP 选项
$mail_contenttype = 'text/plain'; // 邮件内容类型
$mail_wordwrap = 0; // 是否换行（0 表示禁用）
$mail_charset = 'utf-8'; // 邮件字符集
$mail_priority = 3; // 邮件优先级（3 为正常）

// 界面显示选项
$show_menu = true; // 显示菜单
$show_help = false; // 不显示帮助信息
$display_footer = false; // 不显示页脚
```

---

## 步骤 3：启动服务

在 `docker-compose.yml` 文件所在目录运行以下命令

```bash
docker-compose up -d
```

使用浏览器访问以下地址:

- OpenLDAP：`http://<服务器IP>:80`
- Self-Service Password：`http://<服务器IP>:8080`

---

## 步骤 4：测试与验证

1. 在 SSP 登录界面输入 LDAP 用户的邮箱地址，并尝试找回密码。
2. 检查邮件是否正确发送。
3. 确保密码更新在 LDAP 中成功生效。

---

## 常见问题

### 1. 邮件未发送

- 检查 SMTP 配置是否正确。
- 确认网络是否允许访问 SMTP 服务器。

### 2. 密码更新失败

- 确认 LDAP 管理员账号和密码是否正确。
- 检查 OpenLDAP 服务日志。

### 3. 审计日志为空

- 检查 `ssp.conf.php` 中的日志路径是否正确。

---

至此，您已成功部署了基于 Docker 的 LDAP 与密码自助找回服务！
