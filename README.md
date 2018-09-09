# Socat VPN

这是一个另类的 VPN 方案，仅用于研究，不建议上生产环境。

当前仅用于 IPv4 环境。

## 预备

### 服务端

首先检查 ip_forward 是否开启：

```bash
sysctl net.ipv4.ip_forward
```

运行上面的命令，如果返回

> net.ipv4.ip_forward = 0

表示未开启，使用以下命令可以临时开启：

```bash
sysctl net.ipv4.ip_forward=1
```

然后检查下 iptable 的 DNAT rule 是否已经设置：

```bash
iptables -t nat -L POSTROUTING -v
```

假设网络的出口设备名是 `eth0`，如果显示是里有 target 是 `MASQUERADE`、out 为 `eth0` 表示已经加了，如果没有加的，可以用以下命令临时加上：

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### 客户端（本地）

当前还不支持网络的出口设备 ip 为公网 ip 的，即：

```bash
ip route list
```

显示的 `default` 路由的 `via` 后面是公网 ip。


## 安装

### 安装 socat

Arch:

```bash
pacman -Sy socat
```

Debian/Ubuntu:

```bash
apt install -y socat
```

Centos:

```bash
yum install -y socat
```

### 服务端

把 `socat-vpn-server.service` 复制到 `/etc/systemd/system/` 下，或者直接下载：

```bash
curl -sSL https://raw.githubusercontent.com/zbinlin/socat-vpn/master/socat-vpn-server.service | sudo tee /etc/systemd/system/socat-vpn-server.service >/dev/null
```

### 客户端

把 `socat-vpn-client.service` 复制到 `/etc/systemd/system/` 下，或者直接下载：

```bash
curl -sSL https://raw.githubusercontent.com/zbinlin/socat-vpn/master/socat-vpn-client.service | sudo tee /etc/systemd/system/socat-vpn-client.service >/dev/null
```

## 配置

首先在 `/etc` 下创建一个 `socat-vpn` 的目录：

```bash
sudo mkdir -p /etc/socat-vpn
```

然后把 `config.example/env` 复制到 `/etc/socat-vpn` 下，或者直接下载：

```bash
curl -sSL https://raw.githubusercontent.com/zbinlin/socat-vpn/master/config.example/env | sudo tee /etc/socat-vpn/env >/dev/null
```

修改 `/etc/socat-vpn/env`，把 `HOSTNAME` 改成服务器的 ip 地址或域名。

默认配置服务器的端口为 TCP `8443`，如果该端口被防火墙封了，需要解封或换个端口。

### 配置证书

服务端：

```bash
openssl genrsa 2048 2>/dev/null | \
sudo tee /etc/socat-vpn/server.key | \
openssl req -new -key - -x509 -days 3653 \
  -subj '/CN=127.0.0.1' \
  -config <( \
    printf "[req]\ndistinguished_name = RDN\n[RDN]" \
  ) | \
sudo tee /etc/socat-vpn/server.crt
```

运行上面的命令后，会在 `/etc/socat-vpn` 生成 `server.key` 和 `server.crt`，同时把 `server.crt` 内容输出到终端，这时，要输出的 `server.crt` 内容保存在本地的 `/etc/socat-vpn/client-ca.pem` 里。

如果运行命令时报以下错误：

>Error opening Private Key -
>139883511314320:error:02001002:system library:fopen:No such file or directory:bss_file.c:402:fopen('-','r')
>139883511314320:error:20074002:BIO routines:FILE_CTRL:system lib:bss_file.c:404:

可以使用以下命令：

```bash
openssl req -new -key <(openssl genrsa 2048 2>/dev/null | sudo tee /etc/socat-vpn/server.key) -x509 -days 3653 \
  -subj '/CN=127.0.0.1' \
  -config <( \
    printf "[req]\ndistinguished_name = RDN\n[RDN]" \
  ) | \
sudo tee /etc/socat-vpn/server.crt
```


客户端

```bash
openssl genrsa 2048 2>/dev/null | \
sudo tee /etc/socat-vpn/client.key | \
openssl req -new -key - -x509 -days 3653 \
  -subj '/CN=127.0.0.1' \
  -config <( \
    printf "[req]\ndistinguished_name = RDN\n[RDN]" \
  ) | \
sudo tee /etc/socat-vpn/client.crt
```

差不多的命令，同样，把输出的 `client.crt` 内容保存到服务端 `/etc/socat-vpn/server-ca.pem` 里。


## 运行

## 服务端

使用 `systemctl start socat-vpn-server.service`，设置开机启动：`systemctl enable socat-vpn-client.service`

### 客户端

使用 `systemctl start socat-vpn-client.service` 启动。


**先启动服务端再启动客户端，要不客户端启动时连接不上会报错。**


## 限制

服务端当前仅能支持一个客户端在线
