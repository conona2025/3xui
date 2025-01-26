# 3xui 无奈的选择

使用面板本身就是一种无奈的选择，不会配置。

看 RPRX https://github.com/XTLS/Xray-core/pull/3884 所说的，确实非常正确。

官方也好，教程也好，教会人的只有结果，但过程全程是裸奔的，不会深入下去，RPRX 所说的确实是一种挖坑，养肥在杀。

这个世界是危险的。需要全程匿名。不能有信息的泄漏。

需要科学上网 必须具备 iptables 的基本知识 docker 的基本知识 还有 nginx 的基本知识。

全程的配置 一定要 127.0.0.1:2053:2053 不能有任何 公网 IP 暴露在网络之上。

一旦使用 hosts 就是危险。 开源 不等于 安全。随时都会有后门被植入。

```
      - "50001:50001"
      - "50002:50002"
```

这是需要使用节点的端口，提前配置。

3xui 目前所知道的潜在想法

1. key 的问题
2. 是否会定时在 2053 端口网站访问某个网站，即使不把公网IP 暴露出来，本身也是潜在的危险。任何面板都会有这个潜在的问题（被收买）
3. 明面上的问题是 好像所有的面板 都会故意使用 错误的 hosts 网络模式，好像就是为了轻松，而不会教人定义端口。
    
## 镜像

```
docker pull ghcr.io/mhsanaei/3x-ui:latest
```

## docker compose

```
services:
  3x-ui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3x-ui
    hostname: why
    volumes:
      - ./db/:/etc/x-ui/
      - ./cert/:/root/cert/
    environment:
      XRAY_VMESS_AEAD_FORCED: "false"
      X_UI_ENABLE_FAIL2BAN: "true"
      TZ: Asia/Shanghai
    tty: true
    ports:
      - "127.0.0.1:2053:2053"
      - "50001:50001"
      - "50002:50002"
    restart: unless-stopped
```

## SSH 端口转发

```
ssh -L localhost:2053:127.0.0.1:2053 -p 2500 root@ip 地址（192.27.12.20）
```

使用这样的方式就可以在本地 打开 这个 面板，而不需要暴露在网络之上。


### iptables 

```
#!/bin/bash

# 清空现有的所有规则
echo "Clearing existing iptables rules..."
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -t mangle -F
iptables -t mangle -X

# 设置默认策略：允许所有传入流量，允许所有出去流量
echo "Setting default policies..."
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

# 允许本机之间的通信
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# 限制每秒最大 SYN 包数(防止端口扫描)
iptables -A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 3 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# 允许端口 22 (SSH) 访问 需要修改掉
echo "SSH"
#iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# http https
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

目前所知的是，一个服务器只暴露3个端口 80 443 和 SSH 别的应该都不能被暴露

### NGINX 屏蔽80 443 端口的暴露

```
    server {
        listen 80;
        server_name 服务器IP;  # 仅匹配此 IP 地址的请求

        return 444;  # 返回 444 错误，直接关闭连接
    }

    server {
        listen 443 ssl;
        server_name 服务器IP(192.27.12.20);  # 仅匹配此 IP 地址的 HTTPS 请求

        ssl_certificate /etc/nginx/certs/cert.crt;
        ssl_certificate_key /etc/nginx/certs/key.key;

        return 444;  # 返回 444 错误，直接关闭连接
    }

```

### 所知的安全

https://github.com/wulabing/xray_docker

代码安全，建议自己配置，但是节点访问国内的网站时，有时会出现一些问题。但问题不大。

其实 懂得 nginx 配合 docker 127.0.0.1 套上 CDN 基本网络服务暂时安全。 将 BBR3 开启 网络方面基本 OK。
