# Incus Installation and Setup on Debian 13 (Trixie)

本文件记录在 Debian 13 (trixie) 上安装和初始化 Incus 的实际步骤、遇到的问题与解决办法，并列出已验证可用的主流镜像。

## Environment
- OS: Debian GNU/Linux 13 (trixie)
- Kernel: `6.12.57+deb13-cloud-amd64`
- Arch: `x86_64`
- Incus: `6.0.4` (Debian 官方仓库)

## Install Incus
```bash
DEBIAN_FRONTEND=noninteractive apt-get update -y
DEBIAN_FRONTEND=noninteractive apt-get install -y incus
```

## Enable and initialize
优先使用 socket 激活。
```bash
systemctl enable --now incus.socket

# 自动初始化，使用目录型存储
incus admin init --auto --storage-backend=dir

# 验证版本、存储池与网络
incus version
incus storage list
incus network list
```
本机最终状态：
- 存储池：`default` (driver `dir`)
- 网络：`incusbr0` (NAT，示例 `10.70.183.1/24`)

## 镜像源与常见问题

### 1) 官方 images 源下载慢/超时
```
incus launch images:debian/12 deb12
```
可能卡在 rootfs 下载。可改用国内镜像源。

### 2) 清华源 (TUNA) 列表可见但拉取 404
```
incus remote add mirror-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public
incus launch mirror-images:debian/12 deb12
```
报错示例：
```
Error: Failed instance creation: Unable to fetch https://mirrors.tuna.tsinghua.edu.cn/lxc-images/.../incus.tar.xz: resource not found
```
解决办法：切换到其他镜像源（例如南京大学镜像）或使用官方源。

### 3) 南京大学镜像可用
```bash
incus remote add mirror-images https://mirror.nju.edu.cn/lxc-images/ --protocol=simplestreams --public
```

## 已验证可用的主流镜像 (NJU 镜像)
以下镜像已在本机成功启动并运行：
```bash
# Debian 13
incus launch mirror-images:debian/13 deb13

# Rocky Linux 9
incus launch mirror-images:rockylinux/9 rocky9

# CentOS Stream 9
incus launch mirror-images:centos/9-Stream centos9

# Alpine 3.20 (轻量测试)
incus launch mirror-images:alpine/3.20 alpine320
```

验证示例：
```bash
incus exec deb13 -- cat /etc/debian_version
incus exec rocky9 -- cat /etc/redhat-release
incus exec centos9 -- cat /etc/redhat-release
incus exec alpine320 -- cat /etc/alpine-release
```

## 可选：非 root 使用
```bash
usermod -aG incus-admin $USER
# 重新登录或执行 newgrp incus-admin 使组权限生效
```

## 故障排查
- idmap 报错：
  ```
  Error: System doesn't have a functional idmap setup
  ```
  解决办法（示例分配范围，按实际情况调整）：
  ```bash
  echo 'root:100000:65536' | tee -a /etc/subuid >/dev/null
  echo 'root:100000:65536' | tee -a /etc/subgid >/dev/null
  systemctl restart incus.service || systemctl restart incusd.service
  ```
- 网络无 IPv4：确认 `incusbr0` 处于 `CREATED` 并有 IPv4 子网，检查宿主机防火墙/NAT 规则。
- 容器名冲突：`UNIQUE constraint failed: instances.name`，删除旧实例后重试：
  ```bash
  incus delete --force <name>
  ```
