# 自建 RustDesk 远程桌面服务器完整部署与排错指南

本文档旨在提供一个在云服务器（以腾讯云 Ubuntu 为例）上从零开始部署私有 RustDesk 远程桌面服务，并解决常见问题的完整、可复现的操作流程。

参考：https://cloud.tencent.com/developer/article/2503792

## 目录
1.  [准备工作](#1-准备工作)
2.  [部署 RustDesk 服务器](#2-部署-rustdesk-服务器)
3.  [配置服务器防火墙（关键步骤）](#3-配置服务器防火墙关键步骤)
4.  [客户端配置](#4-客户端配置)
5.  [核心问题排错总结](#5-核心问题排错总结)

---

### 1. 准备工作

在开始之前，请确保您拥有：

*   **一台云服务器**: 拥有公网IP及 `root` 或 `sudo` 权限。本文以腾讯云 Ubuntu Server 为例。
*   **安装 Docker**: Docker 是部署服务的基础。
*   **安装 Docker Compose**: 用于编排和管理 RustDesk 的多个服务容器。

**安装命令 (适用于 Ubuntu/Debian):**

```bash
# 1. 安装 Docker
sudo apt-get update
sudo apt-get install -y docker.io

# 2. 安装 Docker Compose (推荐使用独立二进制文件，兼容性最好)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# 3. 验证安装
docker --version
docker-compose --version
```

---

### 2. 部署 RustDesk 服务器

我们将使用 `docker-compose` 来部署 `hbbs` (ID/注册服务器) 和 `hbbr` (中继服务器)。

**第一步：创建工作目录**

```bash
# 创建并进入目录
mkdir -p /opt/rustdesk
cd /opt/rustdesk
```

**第二步：创建 `docker-compose.yml` 文件**
创建一个名为 `docker-compose.yml` 的文件，并将以下 **最终正确配置** 粘贴进去。

```yaml
# /opt/rustdesk/docker-compose.yml

networks:
  rustdesk-net:
    external: false

services:
  hbbs:
    container_name: hbbs
    ports:
      - "21115:21115"
      - "21116:21116"
      - "21116:21116/udp"
      - "21118:21118"
    image: rustdesk/rustdesk-server:latest
    # ############################################################### #
    # ##              ↓↓↓   这是最关键的配置参数   ↓↓↓             ## #
    # ## -r 参数必须是客户端可以访问的公网IP和hbbr的公网端口。       ## #
    # ############################################################### #
    command: hbbs -r 你的服务器公网IP:21117 -k 你的安全密钥
    volumes:
      - ./data:/root # 将数据持久化到宿主机的data目录
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M # 可根据需求调整内存限制

  hbbr:
    container_name: hbbr
    ports:
      - "21117:21117"
      - "21119:21119"
    image: rustdesk/rustdesk-server:latest
    # 此处的密钥必须与 hbbs 中的完全一致
    command: hbbr -k 你的安全密钥
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M # 可根据需求调整内存限制
```
**注意：**
*   请将 `你的服务器公网IP` 替换为实际的公网IP地址。
*   请将 `你的安全密钥` 替换为您自定义的密码（例如 `abc12345`）。

**第三步：启动服务**
在 `/opt/rustdesk` 目录下，运行以下命令：
```bash
sudo docker-compose up -d
```

---

### 3. 配置服务器防火墙（关键步骤）

网络不通是部署失败最常见的原因。防火墙需要配置两层：**云平台安全组** 和 **服务器内部防火墙**。

#### 3.1 云平台安全组 (例如：腾讯云控制台)

登录您的云服务商控制台，找到服务器实例关联的安全组，添加入站规则，**允许**以下所有端口的流量：

*   **TCP 协议**: `21115`, `21116`, `21117`, `21118`, `21119`
*   **UDP 协议**: `21116`

#### 3.2 服务器内部防火墙 (ufw)

对于 Ubuntu 服务器，需要检查其自带的 `ufw` 防火墙。

**第一步：检查 `ufw` 状态**
```bash
sudo ufw status
```
*   如果状态是 `inactive` (不活动)，则无需任何操作。
*   如果状态是 `active` (活动)，为避免两层防火墙策略冲突导致问题，最简单的办法是直接禁用它，由云安全组统一管理。

**第二步：禁用 `ufw` (如果为 `active`)**
```bash
sudo ufw disable
```

---

### 4. 客户端配置

在您的个人电脑（Windows, macOS, Linux）上打开 RustDesk 客户端：

1.  点击客户端ID右侧的 **三个点 "..."**。
2.  选择 **"ID/中继服务器"**。
3.  在设置窗口中填写：
    *   **ID 服务器**: `你的服务器公网IP`
    *   **中继服务器**: **留空** (客户端会自动推断)
    *   **API 服务器**: **留空** (这是商业版功能)
    *   **Key**: `你的安全密钥` (与服务器 `docker-compose.yml` 中设置的完全一致)
4.  点击“确定”保存。

稍等片刻，客户端左下角应显示**绿色圆点**和“**就绪**”状态，表示已成功连接到您的私有服务器。

**特别提醒**：不要点击客户端主界面的“登录”按钮。该功能用于连接官方的商业版服务，与您的私有服务器无关。

---

### 5. 核心问题排错总结

以下是我们在部署过程中遇到的所有问题及其最终解决方案。

| 遇到的问题/报错信息                                          | 根本原因                                                     | 解决方案                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `sudo: docker-compose: command not found`                    | `docker-compose` 未安装或未在系统路径中。                    | 使用 `curl` 命令从 GitHub 下载最新的独立二进制文件到 `/usr/local/bin/docker-compose` 并授予执行权限。 |
| `Error ... Get "https://registry-1.docker.io/v2/" ... i/o timeout` | 服务器无法连接到 Docker Hub。通常是**出站防火墙**限制或网络问题。 | **1. 检查安全组**：确保云平台的安全组允许服务器的出站流量（特别是到 `TCP:443`）。<br>**2. 配置国内镜像**：为 Docker 配置国内镜像加速器（如阿里云、网易）可有效解决此问题。 |
| `无法连接到中继服务器` (但客户端状态为"就绪")                | `hbbs` 向客户端通告了一个错误的中继服务器地址（例如，一个Docker内部地址）。 | **核心修正**：确保 `docker-compose.yml` 文件中 `hbbs` 服务的 `-r` 参数值是**服务器的公网IP和端口**，即 `command: hbbs -r 你的公网IP:21117 ...`。 |
| `Failed host lookup: 'admin.rustdesk.com' ...`               | 在客户端上误点了“登录”按钮，尝试连接RustDesk官方商业服务器。 | **正确操作**：不要使用登录功能。在客户端的“ID/中继服务器”设置中配置好您的私有服务器IP和密钥即可。 |
| 网络连接超时 (用 `telnet` 或 `nc` 测试端口不通)              | 防火墙拦截。最常见的是云安全组**入站规则**未配置，或服务器内部 `ufw` 防火墙开启并拦截了请求。 | **1. 检查安全组**：确认所有所需端口（`21115-21119`）均已在入站规则中放行。<br>**2. 检查内部防火墙**：运行 `sudo ufw status` 并根据需要禁用它 (`sudo ufw disable`)。 |
