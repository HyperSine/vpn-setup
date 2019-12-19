# VPN Stuff -> StrongSwan

## 1. Install `StrongSwan`

```console
$ sudo apt install strongswan libcharon-standard-plugins libcharon-extra-plugins
```

## 2. Let's Encrypt

申请 `Let's Encrypt` 证书：

```console
$ sudo apt-get install letsencrypt
$ letsencrypt certonly --standalone --email <your email> -d <your domain>
```

备注，证书更新：

```console
$ letsencrypt certonly --standalone --renew-by-default -d <your domain>
```

链接、复制证书到 `/etc/ipsec.d/`

```console
$ ln -s /etc/letsencrypt/live/<your domain>/chain.pem /etc/ipsec.d/cacerts/<your domain>.pem
$ ln -s /etc/letsencrypt/live/<your domain>/cert.pem /etc/ipsec.d/certs/<your domain>.pem
$ cp /etc/letsencrypt/live/<your domain>/privkey.pem /etc/ipsec.d/private/<your domain>.pem
```

## 3. 配置 `/etc/ipsec.conf`

```conf
# ipsec.conf - strongSwan IPsec configuration file

# basic configuration

config setup
        # strictcrlpolicy=yes
        # charondebug="ike 1, knl 1, cfg 0"
        # charondebug="cfg 2, dmn 2, ike 2, net 2"
        uniqueids = no

# Add connections here.

# Sample VPN connections

#conn sample-self-signed
#      leftsubnet=10.1.0.0/16
#      leftcert=selfCert.der
#      leftsendcert=never
#      right=192.168.0.2
#      rightsubnet=10.2.0.0/16
#      rightcert=peerCert.der
#      auto=start

#conn sample-with-ca-cert
#      leftsubnet=10.1.0.0/16
#      leftcert=myCert.pem
#      right=192.168.0.2
#      rightsubnet=10.2.0.0/16
#      rightid="C=CH, O=Linux strongSwan CN=peer name"
#      auto=start

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes

    ike=aes256-sha1-modp1024,aes128-sha1-modp1024,3des-sha1-modp1024!
    esp=aes256-sha256,aes256-sha1,3des-sha1!

    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid=@<your domain>
    leftcert=<your domain ssl certificate>
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid=%any
    rightauth=eap-mschapv2
    rightdns=8.8.8.8,8.8.4.4
    rightsourceip=10.239.240.0/24
    rightsendcert=never
    eap_identity=%identity
```

## 4. 配置 `/etc/ipsec.secrets`

```conf
# This file holds shared secrets or RSA private keys for authentication.
# RSA private key for this host, authenticating it to any other host
# which knows the public part.

dev.doublesine.net : RSA "/etc/letsencrypt/live/<your domain>/privkey.pem"

<username> %any% : EAP "<password>"
```

## 5. 系统配置

1. 防火墙开放端口：

   ```console
   $ sudo ufw allow 500,4500/udp
   ```

2. 防火墙允许转发：

   修改 `/etc/ufw/before.rules`， 在 `*filter` 前添加：（注意 `ens3` 换成自己的网卡标识符）

   ```conf
   *nat
   -A POSTROUTING -s 10.239.240.0/24 -o ens3 -m policy --pol ipsec --dir out -j ACCEPT
   -A POSTROUTING -s 10.239.240.0/24 -o ens3 -j MASQUERADE
   COMMIT

   *mangle
   -A FORWARD --match policy --pol ipsec --dir in -s 10.239.240.0/24 -o ens3 -p tcp -m tcp --tcp-flags SYN,RST SYN -m tcpmss --mss 1361:1536 -j TCPMSS --set-mss 1360
   COMMIT
   ```

   在 `*filter` 后添加：

   ```conf
   -A ufw-before-forward --match policy --pol ipsec --dir in --proto esp -s 10.239.240.0/24 -j ACCEPT
   -A ufw-before-forward --match policy --pol ipsec --dir out --proto esp -d 10.239.240.0/24 -j ACCEPT
   ```

3. 修改 `/etc/ufw/sysctl.conf`

   ```conf
   # Enable forwarding
   # Uncomment the following line
   net/ipv4/ip_forward=1

   # Do not accept ICMP redirects (prevent MITM attacks)
   # Ensure the following line is set
   net/ipv4/conf/all/accept_redirects=0

   # Do not send ICMP redirects (we are not a router)
   # Add the following lines
   net/ipv4/conf/all/send_redirects=0
   net/ipv4/ip_no_pmtu_disc=1
   ```

4. 重启 `ufw`

   ```console
   $ sudo ufw disable
   $ sudo ufw enable
   ```

