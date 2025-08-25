# 使用Docker运行iStoreOS并设置旁路由指南

本文档详细介绍如何下载提取的iStoreOS rootfs，使用Docker启动，并配置为旁路由模式。

## 📋 目录

- [系统要求](#系统要求)
- [下载rootfs](#下载rootfs)
- [构建Docker镜像](#构建docker镜像)
- [运行iStoreOS容器](#运行istoreos容器)
- [配置旁路由](#配置旁路由)
- [网络配置](#网络配置)
- [常用操作](#常用操作)
- [故障排除](#故障排除)

## 🔧 系统要求

### 硬件要求
- **架构**: x86_64或ARM64
- **内存**: 至少512MB可用内存
- **存储**: 至少2GB可用磁盘空间
- **网络**: 至少一个网络接口

### 软件要求
- **Docker**: 20.10.0+
- **Docker Compose**: 1.29.0+（可选）
- **操作系统**: Linux发行版（推荐Ubuntu 20.04+）

### 检查系统兼容性
```bash
# 检查Docker版本
docker --version

# 检查系统架构
uname -m

# 检查可用内存
free -h

# 检查网络接口
ip addr show
```

## 📥 下载rootfs

### 方法1: 从GitHub Releases下载

```bash
# 创建工作目录
mkdir -p ~/istoreos-docker
cd ~/istoreos-docker

# 下载最新版本的rootfs（替换为实际的release URL）
wget https://github.com/YOUR_USERNAME/istore-arm-rootfs/releases/latest/download/rootfs.tar.gz

# 验证下载的文件
ls -lh rootfs.tar.gz
```

### 方法2: 使用curl下载

```bash
# 使用curl下载
curl -L -o rootfs.tar.gz https://github.com/YOUR_USERNAME/istore-arm-rootfs/releases/latest/download/rootfs.tar.gz

# 检查文件完整性（如果有提供SHA256）
sha256sum rootfs.tar.gz
```

## 🐳 构建Docker镜像

### 创建Dockerfile

```bash
# 创建Dockerfile
cat > Dockerfile << 'EOF'
FROM scratch
LABEL maintainer="iStoreOS Docker Image"
LABEL description="iStoreOS ARM64 RootFS Docker Image"

# 添加rootfs内容
ADD rootfs.tar.gz /

# 设置环境变量
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV TERM=xterm

# 创建必要的目录
RUN mkdir -p /tmp /var/lock /var/run

# 设置入口点
CMD ["/sbin/init"]
EOF
```

### 构建镜像

```bash
# 构建Docker镜像
docker build -t istoreos:latest .

# 查看构建的镜像
docker images | grep istoreos
```

### 使用多阶段构建（推荐）

```bash
# 创建优化的Dockerfile
cat > Dockerfile.optimized << 'EOF'
# 第一阶段：准备rootfs
FROM alpine:latest as prep
RUN apk add --no-cache tar
WORKDIR /rootfs
ADD rootfs.tar.gz .

# 修复权限和创建必要目录
RUN chmod 755 /rootfs && \
    mkdir -p /rootfs/tmp /rootfs/var/lock /rootfs/var/run /rootfs/sys /rootfs/proc /rootfs/dev

# 第二阶段：最终镜像
FROM scratch
COPY --from=prep /rootfs /
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV TERM=xterm
CMD ["/sbin/init"]
EOF

# 使用优化的Dockerfile构建
docker build -f Dockerfile.optimized -t istoreos:optimized .
```

## 🚀 运行iStoreOS容器

### 基本运行模式

```bash
# 基本运行（仅用于测试）
docker run -it --rm istoreos:latest /bin/sh
```

### 旁路由模式运行

```bash
# 创建macvlan网络（替换为您的网络配置）
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

# 运行iStoreOS容器作为旁路由
docker run -d \
  --name istoreos-gateway \
  --privileged \
  --network macvlan-net \
  --ip 192.168.1.100 \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_ADMIN \
  --device /dev/net/tun \
  --sysctl net.ipv4.ip_forward=1 \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  --restart unless-stopped \
  -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
  istoreos:latest /sbin/init
```

### 使用Docker Compose（推荐）

```bash
# 创建docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  istoreos:
    build: .
    container_name: istoreos-gateway
    privileged: true
    restart: unless-stopped
    networks:
      macvlan-net:
        ipv4_address: 192.168.1.100
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
    devices:
      - /dev/net/tun
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
      - istoreos-data:/overlay
    command: /sbin/init

networks:
  macvlan-net:
    driver: macvlan
    driver_opts:
      parent: eth0  # 替换为您的网络接口
    ipam:
      config:
        - subnet: 192.168.1.0/24
          gateway: 192.168.1.1

volumes:
  istoreos-data:
    driver: local
EOF

# 启动服务
docker-compose up -d

# 查看状态
docker-compose ps
```

## ⚙️ 配置旁路由

### 1. 进入容器配置

```bash
# 进入运行中的容器
docker exec -it istoreos-gateway /bin/sh

# 或者使用ash（如果available）
docker exec -it istoreos-gateway /bin/ash
```

### 2. 网络接口配置

```bash
# 在容器内执行以下命令

# 配置网络接口
uci set network.lan.proto='static'
uci set network.lan.ipaddr='192.168.1.100'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='8.8.8.8 8.8.4.4'

# 提交配置
uci commit network

# 重启网络服务
/etc/init.d/network restart
```

### 3. 防火墙配置

```bash
# 配置防火墙规则
uci set firewall.@defaults[0].forward='ACCEPT'
uci set firewall.@zone[0].input='ACCEPT'
uci set firewall.@zone[0].output='ACCEPT'
uci set firewall.@zone[0].forward='ACCEPT'

# 提交防火墙配置
uci commit firewall

# 重启防火墙
/etc/init.d/firewall restart
```

### 4. DHCP服务配置

```bash
# 禁用DHCP服务（避免与主路由冲突）
uci set dhcp.lan.ignore='1'
uci commit dhcp
/etc/init.d/dnsmasq restart

# 或者配置DHCP中继
uci set dhcp.lan.ignore='0'
uci set dhcp.lan.start='150'
uci set dhcp.lan.limit='50'
uci set dhcp.lan.leasetime='12h'
uci commit dhcp
```

## 🌐 网络配置

### 主机网络配置

在运行Docker的主机上，您需要配置网络以支持旁路由：

```bash
# 1. 创建网桥接口（如果使用bridge模式）
sudo ip link add name br-lan type bridge
sudo ip link set br-lan up

# 2. 配置路由规则（将特定流量导向iStoreOS）
sudo ip route add 192.168.1.100/32 dev docker0

# 3. 配置iptables规则（如需要）
sudo iptables -I FORWARD -i docker0 -j ACCEPT
sudo iptables -I FORWARD -o docker0 -j ACCEPT
```

### 客户端设备配置

在需要使用旁路由的设备上：

```bash
# 方法1: 修改默认网关
sudo ip route del default
sudo ip route add default via 192.168.1.100

# 方法2: 添加特定路由
sudo ip route add 0.0.0.0/1 via 192.168.1.100
sudo ip route add 128.0.0.0/1 via 192.168.1.100

# 方法3: 修改DNS设置
echo "nameserver 192.168.1.100" | sudo tee /etc/resolv.conf
```

## 🔧 常用操作

### 容器管理

```bash
# 查看容器状态
docker ps -a

# 查看容器日志
docker logs istoreos-gateway

# 重启容器
docker restart istoreos-gateway

# 停止容器
docker stop istoreos-gateway

# 删除容器
docker rm istoreos-gateway
```

### 进入Web管理界面

```bash
# 如果iStoreOS包含Web界面
# 访问: http://192.168.1.100
# 默认用户名: root
# 默认密码: （通常为空或password）

# 在容器内修改密码
docker exec -it istoreos-gateway passwd root
```

### 安装软件包

```bash
# 进入容器
docker exec -it istoreos-gateway /bin/sh

# 更新软件包列表
opkg update

# 安装软件包
opkg install luci-app-ssr-plus
opkg install luci-app-openclash

# 查看已安装包
opkg list-installed
```

### 数据备份

```bash
# 导出容器数据
docker export istoreos-gateway > istoreos-backup.tar

# 备份配置文件
docker exec istoreos-gateway tar czf - /etc > config-backup.tar.gz

# 备份整个容器卷
docker run --rm -v istoreos-data:/data -v $(pwd):/backup busybox tar czf /backup/data-backup.tar.gz /data
```

## 🔍 故障排除

### 常见问题

#### 1. 容器无法启动

```bash
# 检查Docker服务状态
sudo systemctl status docker

# 检查镜像是否存在
docker images | grep istoreos

# 查看详细错误信息
docker logs istoreos-gateway
```

#### 2. 网络连接问题

```bash
# 检查容器网络配置
docker exec istoreos-gateway ip addr show

# 检查路由表
docker exec istoreos-gateway ip route show

# 测试网络连通性
docker exec istoreos-gateway ping 8.8.8.8
```

#### 3. 权限问题

```bash
# 确保容器以特权模式运行
docker inspect istoreos-gateway | grep Privileged

# 检查capabilities
docker inspect istoreos-gateway | grep CapAdd
```

#### 4. 性能问题

```bash
# 查看容器资源使用
docker stats istoreos-gateway

# 限制容器资源
docker update --memory=512m --cpus=1.0 istoreos-gateway
```

### 调试命令

```bash
# 进入容器进行调试
docker exec -it istoreos-gateway /bin/sh

# 查看系统进程
ps aux

# 查看网络状态
netstat -tuln

# 查看系统日志
logread

# 查看内核日志
dmesg
```

### 重置配置

```bash
# 停止容器
docker stop istoreos-gateway

# 删除容器和卷
docker rm istoreos-gateway
docker volume rm istoreos-data

# 重新创建并启动
docker-compose up -d
```

## 📚 高级配置

### 自动启动脚本

创建系统服务来自动管理iStoreOS容器：

```bash
# 创建systemd服务文件
sudo tee /etc/systemd/system/istoreos-docker.service << 'EOF'
[Unit]
Description=iStoreOS Docker Container
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/user/istoreos-docker
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

# 启用服务
sudo systemctl enable istoreos-docker.service
sudo systemctl start istoreos-docker.service
```

### 监控和日志

```bash
# 设置日志轮转
docker exec istoreos-gateway logrotate /etc/logrotate.conf

# 监控脚本
cat > monitor.sh << 'EOF'
#!/bin/bash
while true; do
    if ! docker ps | grep -q istoreos-gateway; then
        echo "$(date): iStoreOS container is down, restarting..."
        docker-compose restart
    fi
    sleep 30
done
EOF

chmod +x monitor.sh
nohup ./monitor.sh &
```

## 📝 注意事项

1. **网络配置**: 确保容器的IP地址不与现有设备冲突
2. **防火墙**: 可能需要调整主机防火墙规则
3. **性能**: 容器模式可能比原生运行性能略低
4. **存储**: 使用Docker卷来持久化配置数据
5. **更新**: 定期更新rootfs以获得最新功能和安全补丁

## 🆘 获取帮助

- **iStoreOS官方文档**: [https://www.istoreos.com/](https://www.istoreos.com/)
- **Docker官方文档**: [https://docs.docker.com/](https://docs.docker.com/)
- **OpenWrt Wiki**: [https://openwrt.org/](https://openwrt.org/)

---

*本指南基于iStoreOS ARM64 rootfs和Docker容器技术。根据您的具体网络环境，可能需要调整相关配置。*