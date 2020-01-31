# VPN Stuff -> OpenConnect

## 1. Install `ocserv`

```console
$ sudo apt install ocserv
```

## 2. Configuare

```console
$ sudo vim /etc/ocserv/ocserv.conf
```

1. 使用密码认证：

   ```
   auth = "plain[passwd=/etc/ocserv/ocpasswd]"
   ```

2. 更改监听端口：

   ```
   tcp-port = 443
   udp-port = 443
   ```

3. 设置SSL证书：

   ```
   server-cert = /etc/letsencrypt/live/<your domain>/fullchain.pem
   server-key = /etc/letsencrypt/live/<your domain>/privkey.pem
   ```

4. 修改最大客户端数：

   ```
   max-clients = 0  # 0为无限制
   ```

5. 修改最大同时连接数：

   ```
   max-same-clients = 0  # 0为无限制
   ```

6. 设置域名：

   ```
   default-domain = <your domain>
   ```

8. 设置IPv4网段：

   ```
   ipv4-network = 10.10.10.0
   ipv4-netmask = 255.255.255.0 
   ```

9. 转发所有DNS查询：

   ```
   tunnel-all-dns = true
   ```

10. 设置DNS服务器：

    ```
    dns = 8.8.4.4
    dns = 8.8.8.8
    ```

11. 注释全部路由设置，让服务器作为默认网关：

    ```
    #route = 10.0.0.0/8
    #route = 172.16.0.0/12
    #route = 192.168.0.0/16
    ```

12. 可选：

    ```
    try-mtu-discovery = true
    ```

## 3. System Configuare

1. 开启IP转发

   ```
   $ sudo vim /etc/sysctl.conf
   net.ipv4.ip_forward = 1
   ```

2. `ufw` 默认接受转发：

   `/etc/default/ufw`

   ```conf
   DEFAULT_FORWARD_POLICY="ACCEPT"
   ```

3. `ufw` NAT表：

   `/etc/ufw/before.rules`

   ```conf
   # NAT table rules
   *nat
   -A POSTROUTING -s 10.10.10.0/24 -o ens3 -j MASQUERADE
   ```

4. 如果ocserv监听端口改变不了，修改 `/lib/systemd/system/ocserv.service`，注释掉下面两行：

   ```conf
   [Unit]
   ...
   #Requires=ocserv.socket

   [Service]
   ...
   ...

   [Install]
   ...
   #Also=ocserv.socket
   ```

   同时关闭 `ocserv.socket` 服务：

   ```console
   $ sudo systemctl disable ocserv.socket
   ```


## 4. Add VPN account

```console
$ sudo ocpasswd -c /etc/ocserv/ocpasswd <new username>
```
