## 一、准备环境

### 1️⃣ 安装 Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable docker --now
```

### 2️⃣ 安装 Docker Compose（新版）

```bash
docker compose version
```

如果没有：

```bash
sudo apt install -y docker-compose-plugin
```

---

## 二、选择 go-socks5-proxy 镜像

常用、稳定的有两个：

### ✅ 方案一（推荐）：`serjs/go-socks5-proxy`

* 支持用户名密码
* 轻量、简单

Docker Hub：

```
serjs/go-socks5-proxy
```

---

## 三、目录结构

```bash
mkdir -p /opt/socks5
cd /opt/socks5
```

---

## 四、docker-compose.yml 示例

### 🔐 带用户名密码的 SOCKS5

```yaml
version: "3.8"

services:
  socks5:
    image: serjs/go-socks5-proxy
    container_name: socks5
    restart: unless-stopped
    ports:
      - "1080:1080"
    environment:
      - PROXY_USER=myuser
      - PROXY_PASSWORD=mypassword
```

📌 说明：

* 宿主机端口：`1080`
* SOCKS5 地址：`服务器IP:1080`
* 认证方式：用户名密码

---

### 🚫 不带认证（不推荐公网）

```yaml
version: "3.8"

services:
  socks5:
    image: serjs/go-socks5-proxy
    container_name: socks5
    restart: unless-stopped
    ports:
      - "1080:1080"
```

⚠️ **仅建议内网使用**

---

## 五、启动服务

```bash
docker compose up -d
```

查看状态：

```bash
docker compose ps
docker logs socks5
```

---

## 六、防火墙放行端口（如果有）

### UFW

```bash
sudo ufw allow 1080/tcp
```

### iptables

```bash
iptables -I INPUT -p tcp --dport 1080 -j ACCEPT
```

---

## 七、客户端测试

### curl 测试

```bash
curl --socks5 myuser:mypassword@127.0.0.1:1080 http://ipinfo.io
```

### 浏览器

* SOCKS5
* 地址：服务器 IP
* 端口：1080
* 启用用户名密码

---

## 八、进阶（可选）

### ✔️ 限制监听内网

```yaml
ports:
  - "127.0.0.1:1080:1080"
```

### ✔️ 使用 host 网络（低延迟）

```yaml
network_mode: host
```

### ✔️ 多账号（需要换镜像或自定义）

还可以：

* **TLS + SOCKS5**
* **IPv6**
* **限速 / 并发控制**
* **只允许指定 IP 连接**
* **透明代理 / 与 VPN 联动**
