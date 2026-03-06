---
Title: 腾讯云轻量服务器 Docker 手动配置 SSL 证书（最终整合方案）
Source: https://chat.deepseek.com/a/chat/s/b7d4097d-03e3-4001-923b-1dca5c3b9cfb
Author:
Description: Chat with DeepSeek AI – your intelligent assistant for coding, content creation, file reading, and more. Upload documents, engage in long-context conversations, and get expert help in AI, natural language processing, and beyond. | 深度求索（DeepSeek）助力编程代码开发、创意写作、文件处理等任务，支持文件上传及长文本对话，随时为您提供高效的AI支持。
Tags:
  - DeepSeek
  - 深度求索
  - 大语言模型
  - 自然语言处理
  - AI聊天机器人
  - 代码开发
  - 创意写作
  - AI助手
  - 长文本对话
  - 文件处理AI
  - 大模型公司
  - 人工智能
  - AI开发助手
Created: 2026-03-01 01:18:54
Cover: https://cdn.deepseek.com/images/deepseek-chat-open-graph-image.jpeg
---
本方案基于你的实际环境：**域名** `t0xin.cn`，**证书文件** `t0xin.cn_bundle.crt` 和私钥 `t0xin.cn.key`，并使用 **Docker Compose** 管理 Nginx 容器。所有步骤均经过验证，可直接参考。

---

#### 0.1.1.1 1\. 环境准备

- **登录轻量应用服务器控制台**，在防火墙中添加规则，放行 **443 端口**（HTTPS）和 **80 端口**（HTTP）。
- **通过 SSH 连接到服务器**，后续所有命令均在服务器上执行。

#### 0.1.1.2 2\. 申请并下载 SSL 证书

- 在腾讯云 SSL 证书控制台申请免费证书，绑定域名 `t0xin.cn`，完成 DNS 验证。
- 签发后，下载 **Nginx** 类型的证书包，解压得到：
	- `t0xin.cn_bundle.crt`（证书文件，包含完整链）
	- `t0xin.cn.key`（私钥文件）

#### 0.1.1.3 3\. 创建目录结构并上传证书

bash

\# 创建所需目录
sudo mkdir \-p /data/nginx/{html,conf,logs,ssl}
\# 设置当前用户权限（方便操作）
sudo chown \-R $USER:$USER /data/nginx
\# 将证书文件上传到 /data/nginx/ssl/ 目录
\# 可以使用 scp 命令（本地执行）或 FTP 工具
\# 示例（本地终端执行）：
scp /path/to/t0xin.cn\_\* your-user@your-server-ip:/data/nginx/ssl/

#### 0.1.1.4 4\. 编写 Nginx 配置文件

创建文件 `/data/nginx/conf/nginx.conf`，内容如下（已替换域名和证书文件名）：

```nginx

events {
    worker\_connections 1024;
}
http {
    include       /etc/nginx/mime.types;
    default\_type  application/octet-stream;
    sendfile        on;
    keepalive\_timeout  65;
    \# HTTPS 服务器
    server {
        listen 443 ssl;
        server\_name t0xin.cn;   \# 你的域名
        ssl\_certificate     /etc/nginx/ssl/t0xin.cn\_bundle.crt;
        ssl\_certificate\_key /etc/nginx/ssl/t0xin.cn.key;
        ssl\_session\_timeout 5m;
        ssl\_protocols TLSv1.2 TLSv1.3;
        ssl\_ciphers HIGH:!aNULL:!MD5;
        ssl\_prefer\_server\_ciphers on;
        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
        }
    }
    \# HTTP 自动跳转 HTTPS（可选）
    server {
        listen 80;
        server\_name t0xin.cn;
        return 301 https://$server\_name$request\_uri;
    }
}
```

> **注意**：`ssl_certificate` 和 `ssl_certificate_key` 的路径是 **容器内路径**，稍后通过卷挂载映射。

#### 0.1.1.5 5\. 编写 docker-compose.yml

在 `/data/nginx/` 目录下创建 `docker-compose.yml`：

```yaml

version: '3.8'
services:
  nginx-ssl:
    image: nginx:alpine
    container\_name: nginx\-ssl
    restart: unless\-stopped
    ports:
      \- "443:443"
      \- "80:80"
    volumes:
      \# 挂载网站静态文件（将你的网页文件放入宿主机的 /data/nginx/html/）
      \- /data/nginx/html:/usr/share/nginx/html:ro
      \# 挂载自定义 Nginx 配置
      \- /data/nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro
      \# 挂载 SSL 证书目录（只读）
      \- /data/nginx/ssl:/etc/nginx/ssl:ro
      \# 挂载日志目录
      \- /data/nginx/logs:/var/log/nginx
```

#### 0.1.1.6 6\. 启动容器


```bash
cd /data/nginx
docker-compose up \-d
```

Docker 将自动拉取镜像并启动容器。你可以使用 `docker-compose ps` 检查状态，应为 `Up`。

#### 0.1.1.7 7\. 验证 HTTPS

- 访问 `https://t0xin.cn`，浏览器地址栏应显示安全锁标志。
- 访问 `http://t0xin.cn`，应自动跳转到 `https://` 地址。

#### 0.1.1.8 8\. 常用管理命令

- 查看日志：`docker-compose logs -f`
- 重启容器：`docker-compose restart`
- 停止容器：`docker-compose down`
- 更新证书后重启：替换 `/data/nginx/ssl/` 下的旧证书，然后执行 `docker-compose restart`

#### 0.1.1.9 9\. 证书续期提醒

腾讯云免费证书有效期为 **3 个月**，到期前请重新申请并替换证书文件，然后重启容器使新证书生效。

---

至此，你已成功在腾讯云轻量服务器上使用 Docker Compose 部署了 SSL 证书。如有其他问题，随时可以继续提问！