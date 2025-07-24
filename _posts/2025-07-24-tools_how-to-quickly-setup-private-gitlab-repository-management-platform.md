---
title: 如何快速搭建私有的 GitLab 代码管理平台
date: 2025-07-24 17:38:00 +0800
categories: [工具]
tags: [GitLab, git]
pin: false
---

## 介绍

在本教程中，您将学习如何使用 Docker 和 Docker Compose 来搭建 GitLab 代码仓库管理平台，并配置 SSL 加密与 LDAP 认证。GitLab 是一个流行的 Git 仓库管理工具，广泛用于源代码管理与团队协作。本教程将帮助您在Linux环境中轻松搭建一个私有 GitLab 服务，并通过 LDAP 实现集中的用户认证。

## 本教程的内容

1. **环境准备**
    - 安装 Docker 和 Docker Compose
    - 设置 GitLab 所需的端口与文件路径
2. **配置 GitLab 服务**
    - 使用 Docker Compose 安装 GitLab
    - 配置 SSL 加密
    - 配置 LDAP 认证
3. **配置 SMTP 邮件发送**
4. **启动 GitLab 服务并访问**
    - 启动 GitLab 容器
    - 登录 GitLab 并验证配置
5. **（可选）配置 GitLab 的 Git SSH 功能**

## 预备知识

- 基础的 Linux 系统操作
- Docker 与 Docker Compose 的基础知识
- 对 GitLab、LDAP 和 SMTP 有基本了解

## 环境准备

在开始之前，您需要准备一个 Linux 服务器并安装 Docker 和 Docker Compose。可以通过以下命令进行安装：

```bash
# 安装 Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo DOWNLOAD_URL=https://mirrors.ustc.edu.cn/docker-ce sh get-docker.sh

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

## 配置 GitLab 服务

我们将使用 Docker Compose 来启动 GitLab 服务，确保 Docker 容器能够持久存储数据和配置。

### 1. 创建 `docker-compose.yml` 配置文件

在您的服务器上，创建一个新的目录用于存放 GitLab 的配置和数据。进入该目录并创建 `docker-compose.yml` 文件。

```bash
mkdir ~/gitlab
cd ~/gitlab
nano docker-compose.yml
```

将以下内容粘贴到 `docker-compose.yml` 文件中：

```yaml
version: '3'
services:
  gitlab:
    image: 'gitlab/gitlab-ce:latest' # 或者是 gitlab-ce 版本
    container_name: 'gitlab'
    restart: always
    hostname: 'gitlab.algs.tech'
    ports:
      - '80:80'
      - '443:443'
      - '22:22'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.algs.tech'
        nginx['ssl_certificate'] = "/etc/gitlab/ssl/cert.pem"
        nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/key.pem"
        gitlab_rails['ldap_enabled'] = true
        gitlab_rails['ldap_servers'] = YAML.load <<-EOS
          main:
            label: 'LDAP'
            host: 'openldap.algs.tech'  # 替换为您的 LDAP 服务器地址
            port: 389  # LDAP 端口
            uid: 'uid'  # LDAP 用户名属性
            bind_dn: 'cn=admin,dc=webcoding,dc=tech'  # LDAP 管理员 DN
            password: 'abc2024XYZ'  # LDAP 管理员密码
            encryption: 'plain'  # 如果需要加密，改为 'start_tls' 或 'simple_tls'
            verify_certificates: true  # 如果使用 TLS 并需要跳过证书验证，设置为 false
            active_directory: true  # 如果使用 Active Directory，设置为 true
            allow_username_or_email_login: false  # 是否允许使用用户名或邮件登录
            lowercase_usernames: false
            block_auto_created_users: false
            base: 'dc=webcoding,dc=tech'  # LDAP 基础 DN
            user_filter: ''  # 额外的用户过滤条件
            group_base: ''
            admin_group: ''  # 可以设置管理员组
            sync_ssh_keys: false
        EOS
        # SMTP 配置
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "smtphz.qiye.163.com"  # SMTP 服务器地址
        gitlab_rails['smtp_port'] = 465  # SMTP 端口
        gitlab_rails['smtp_user_name'] = "noreply.gitlab@aiapp.pro"  # 发件邮箱
        gitlab_rails['smtp_password'] = "password"  # 发件邮箱密码
        gitlab_rails['smtp_domain'] = "qiye.163.com"  # 域名
        gitlab_rails['smtp_authentication'] = "login"  # 认证方式，可选"plain", "login", "cram_md5"
        gitlab_rails['smtp_enable_starttls_auto'] = false  # 启用 STARTTLS
        gitlab_rails['smtp_tls'] = true
        gitlab_rails['smtp_openssl_verify_mode'] = 'peer'  # SSL 验证模式，可选"none", "peer", "client_once", "fail_if_no_peer_cert"
        gitlab_rails['gitlab_email_from'] = 'noreply.gitlab@aiapp.pro'
        user['git_user_email'] = 'noreply.gitlab@aiapp.pro'
    volumes:
      - '$GITLAB_HOME/config:/etc/gitlab'
      - '$GITLAB_HOME/logs:/var/log/gitlab'
      - '$GITLAB_HOME/data:/var/opt/gitlab'
      - '/etc/certs/ssl/algs.tech:/etc/gitlab/ssl'
    shm_size: '256m'
```

### 2. 配置 SSL 证书

确保您已经拥有有效的 SSL 证书和密钥，将它们放置在 `/etc/certs/ssl/algs.tech/` 目录下，并且它们的权限设置正确。

### 3. 启动 GitLab

在配置好 Docker Compose 文件后，您可以使用以下命令启动 GitLab 服务：

```bash
docker-compose up -d
```

此命令会启动 GitLab 容器并在后台运行。

## 访问 GitLab

1. 打开浏览器并访问 `https://gitlab.algs.tech`。
2. 默认情况下，GitLab 会要求您设置管理员密码。
3. 使用您的管理员账号登录。

## 验证 LDAP 集成

在 GitLab 登录页面，尝试使用 LDAP 用户进行登录。确保您正确配置了 LDAP 地址、端口、管理员 DN 和密码。

## 配置 GitLab 的 Git SSH 功能

如果需要启用 Git 通过 SSH 的方式推送和拉取代码，可以配置 GitLab 使用 SSH。您可以参考 [GitLab 官方文档](https://docs.gitlab.com/ee/user/ssh.html) 了解更多配置内容。

---

### 结语

恭喜！您已经成功搭建了一个内网 GitLab 代码仓库管理平台，并实现了 SSL 加密和 LDAP 集成。通过这个平台，您的团队可以更加安全、高效地进行代码管理。

你可以将这个过程录制成教程视频，并将其发布到 YouTube 或比例比例平台，帮助更多的开发者和 IT 管理员搭建类似的环境。