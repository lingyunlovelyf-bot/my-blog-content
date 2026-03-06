---
Title: nginx-proxy-manager实现反向代理+自动化证书(实战)-腾讯云开发者社区-腾讯云
Source: "https://cloud.tencent.com.cn/developer/article/2527035?policyId=1003"
Author: 
Description: Nginx Proxy Manager 是开源反向代理工具，基于 Nginx 和 Docker，提供 Web 界面管理反向代理、SSL 证书及端口映射，适合个人或团队简化多服务部署。
Tags:
  - nginx-proxy-manager
  - 反向代理
  - SSL证书
  - Docker
Created: "2026-03-01 22:49:16"
Cover: "https://cloudcache.tencent-cloud.com/open_proj/proj_qcloud_v2/gateway/shareicons/cloud.png"
---

#### 0.1.1.1 nginx-proxy-manager是什么

Nginx Proxy Manager 是一个开源的、基于 Web 界面的反向代理管理工具。它的核心是基于 Nginx 和 Docker 的，旨在提供一种简单易用的方式来管理 Nginx 反向代理、[SSL 证书](https://cloud.tencent.com/product/ssl?from_column=20065&from=20065)和各种代理设置。

**主要功能**

​ • **基于 Web 界面的管理**：提供友好的 Web 用户界面，用户无需手动编辑 Nginx 配置文件。

​ • **反向代理**：可以轻松地为多个服务或网站设置反向代理规则，将外部请求根据域名或路径转发到内部的不同服务。

​ • **自动获取 SSL 证书**：集成了 Let’s Encrypt，可以轻松为站点获取和续订免费的 SSL 证书。

​ • **访问控制**：可以通过设置用户名和密码来保护特定的路径或站点。

​ • **端口映射**：支持将请求从外部端口映射到内部服务的不同端口。

**使用场景**

Nginx Proxy Manager 主要用于需要在多个服务之间进行反向代理的场景，比如你有多个 Web 服务（如个人博客、Nextcloud、GitLab 等）需要公开在互联网上，但你希望通过单一的入口来管理这些服务。它可以简化反向代理和 SSL 证书管理，尤其适合以下情况：

​ • 个人或小型团队使用 Docker 部署多个 Web 服务。

​ • 希望通过一个简单的 Web 管理界面来管理所有代理规则和 SSL 证书。

​ • 需要简单的域名到服务的映射，但不想深入研究 Nginx 配置文件。

**特点**

​ • 基于 Docker 容器的部署方式，简单方便。

​ • 集成 Let’s Encrypt，可以自动获取和管理 SSL 证书。

​ • 具有直观的 Web 界面，无需编写复杂的 Nginx 配置文件。

​ • 轻松管理子域、端口转发等功能。

#### 0.1.1.2 搭建nginx-proxy-manager

> 我这里使用的是docker-compose搭建的，比较方便，如果以后迁移[服务器](https://cloud.tencent.com/product/cvm?from_column=20065&from=20065)的时候也比较简单

##### 0.1.1.2.1 搭建

代码语言：javascript

AI代码解释

复制

```javascript
version: '3'
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'       # HTTP
      - '10811:81'       # Nginx Proxy Manager 管理界面
      - '443:443'     # HTTPS
    volumes:
      - ./data:/data  # 数据卷，用于存储 Nginx 配置和证书
      - ./letsencrypt:/etc/letsencrypt  # 证书存储卷

  db:
    image: 'jc21/mariadb-aria:latest'
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: 'ua1wlnkZzzLu7RrB'  # MySQL root 用户密码
      MYSQL_DATABASE: 'npm'                      # NPM 使用的数据库
      MYSQL_USER: 'npm'                          # 数据库用户名
      MYSQL_PASSWORD: 'ua1wlnkZzzLu7RrB'         # 数据库用户密码
    volumes:
      - ./mysql:/var/lib/mysql
```

db这里可以不用也行，看自己

注意：这里我没有使用81端口，因为1000以下的端口在某些情况下会被保护的，比如我使用穿透，就会报错

##### 0.1.1.2.2 成功示意图

![image-20241025104211847](https://t0xin-1315274956.cos.ap-guangzhou.myqcloud.com/smartclip/1772376573289-qarg68.png)

#### 0.1.1.3 实现证书创建加续签

![image-20241025104440164](https://t0xin-1315274956.cos.ap-guangzhou.myqcloud.com/smartclip/1772376573430-mglqzw.png)

它支持一键创建多个证书，而且速度很快，也可以支持一键续订

![续签](https://t0xin-1315274956.cos.ap-guangzhou.myqcloud.com/smartclip/1772376573463-6i8jxn.png)

#### 0.1.1.4 端口转发(两种实现)

##### 0.1.1.4.1 基于本机ip

> 这里的本机ip是ifconfig中的ip，并非127.0.0.1，因为我们的npm是在容器中部署的，所以要想实现通信只能通过本机内网ip实现。

![image-20241025105736249](https://t0xin-1315274956.cos.ap-guangzhou.myqcloud.com/smartclip/1772376573669-ufil3c.png)

##### 0.1.1.4.2 基于docker网络

> 首先创建一个网络，然后让其加入这个网络，就可以直接用容器名称访问了，如下图

![](https://t0xin-1315274956.cos.ap-guangzhou.myqcloud.com/smartclip/1772376573741-08q0ro.png)

#### 0.1.1.5 报错解决

> 如果你的证书被删除了，但是你的转发中存在了该证书，就会报如下错

代码语言：javascript

AI代码解释

复制

```javascript
搭建的nginx proxy manager创建证书的时候报错CommandError: nginx: [warn] “listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/1.conf:19 中改用“http2”指令
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/1.conf:20 中改用“http2”指令
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/2.conf:19 中使用“http2”指令
nginx：[警告] 在 /data/nginx/proxy_host/2.conf:19 中为 0.0.0.0:443 重新定义了协议选项
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/2.conf:20 中改用“http2”指令
nginx：[警告] 在 /data/nginx/proxy_host/2.conf:20 中为 [::]:443 重新定义了协议选项
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/3.conf:19 中使用“http2”指令
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/3.conf:20 中使用“http2”指令
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/4.conf:19 中使用“http2”指令
nginx：[警告]“listen ... http2”指令已弃用，请在 /data/nginx/proxy_host/4.conf:20 中改用“http2”指令
nginx：[emerg] 无法加载证书“/etc/letsencrypt/live/npm-2/fullchain.pem”：BIO_new_file（）失败（SSL：错误：80000002：系统库::没有此文件或目录：调用 fopen（/etc/letsencrypt/live/npm-2/fullchain.pem，r）错误：10000080：BIO 例程::没有此文件）
nginx：配置文件/etc/nginx/nginx.conf 测试失败
    位于 /app/lib/utils.js:16:13
    在 ChildProcess.exithandler （节点：child_process：430：5）
    在 ChildProcess.emit （节点：事件：519：28）
    在可能关闭（节点：内部/子进程：1105：16）
    在 ChildProcess._handle.onexit （节点：internal/child_process：305：5）
```

此时要先停掉转发，在新增创建即可