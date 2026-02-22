OpenConnect VPN (ocserv)
===

## 概要

OpenConnect VPN は Cisco AnyConnect VPN プロトコル互換のオープンソース VPN ソリューションです。  
サーバ側には **ocserv (OpenConnect Server)** を使用します。

- プロトコル: TLS (TCP 443) / DTLS (UDP 443)
- 認証方式: パスワード認証 または 証明書認証

### テスト環境の構成

| ホスト     | OS           | IP アドレス     | 役割                     |
| ---------- | ------------ | --------------- | ------------------------ |
| vpn-server | Ubuntu 24.04 | 192.168.222.145 | ocserv (VPN サーバ)      |
| vpn-client | Ubuntu 24.04 | 192.168.222.146 | OpenConnect クライアント |

- VPN サブネット: `10.255.255.0/24`
- VPN サーバ待受ポート: TCP 443 / UDP 443

---

## 証明書の準備 (サーバ側で実施)

自己認証局 (CA) を作成し、CA で署名したサーバ証明書と (必要に応じて) クライアント証明書を作成します。

### 作業ディレクトリ

```bash
mkdir -p $HOME/ocserv-certs \
    && cd $HOME/ocserv-certs
```

### CA 証明書 (自己認証局) 作成

```bash
openssl req \
    -new \
    -out privateca.crt \
    -keyout privateca.key \
    -x509 \
    -days 3650 \
    -newkey rsa:3072 \
    -nodes \
    -subj "/CN=TestVPN-CA"
```

### サーバ証明書作成

- CSR 作成

    ```bash
    openssl req \
        -new \
        -out server.csr \
        -keyout server.key \
        -newkey rsa:3072 \
        -nodes \
        -subj "/CN=vpn-server.local"
    ```

- Subject Alternative Name の準備

    ```bash
    echo "subjectAltName = DNS:vpn-server.local, IP:192.168.222.145" > san_server.txt
    ```

- CA 署名

    ```bash
    openssl x509 \
        -req \
        -days 3650 \
        -in server.csr \
        -out server.crt \
        -CA privateca.crt \
        -CAkey privateca.key \
        -CAcreateserial \
        -extfile san_server.txt
    ```

- 確認

    ```bash
    openssl x509 -in server.crt -text -noout
    ```

### クライアント証明書作成 (証明書認証を使用する場合)

- CSR 作成

    ```bash
    openssl req \
        -new \
        -out client.csr \
        -keyout client.key \
        -newkey rsa:3072 \
        -nodes \
        -subj "/CN=vpn-user01"
    ```

- CA 署名

    ```bash
    openssl x509 \
        -req \
        -days 3650 \
        -in client.csr \
        -out client.crt \
        -CA privateca.crt \
        -CAkey privateca.key \
        -CAcreateserial
    ```

- PKCS#12 形式ファイルの作成 (クライアントへ配布)

    ```bash
    openssl pkcs12 -export \
        -in client.crt \
        -inkey client.key \
        -certfile privateca.crt \
        -out client.p12 \
        -name "vpn-user01"
    ```

    パスワードを設定する必要があります。

---

## サーバ設定 (ocserv)

### インストール

```bash
sudo apt-get update
sudo apt-get install -y ocserv
```

### 証明書ファイルの配置

```bash
sudo cp $HOME/ocserv-certs/privateca.crt /etc/ocserv/
sudo cp $HOME/ocserv-certs/server.crt    /etc/ocserv/
sudo cp $HOME/ocserv-certs/server.key    /etc/ocserv/
sudo chmod 600 /etc/ocserv/server.key
```

### パスワードファイルの作成 (パスワード認証を使用する場合)

```bash
sudo ocpasswd -c /etc/ocserv/ocpasswd vpn-user01
```

### 設定ファイルの編集

`/etc/ocserv/ocserv.conf` のデフォルトファイルから、最低限変更・確認が必要な項目を以下に示します。

```diff title="/etc/ocserv/ocserv.conf.diff"
--- ocserv.conf.original        2026-02-22 11:03:18.996090605 +0000
+++ ocserv.conf 2026-02-22 12:02:06.850188119 +0000
@@ -48,7 +48,9 @@
 #auth = "pam"
 #auth = "pam[gid-min=1000]"
 #auth = "plain[passwd=/etc/ocserv/passwd,otp=./sample.otp]"
-auth = "plain[passwd=/etc/ocserv/passwd]"
+# パスワード認証 (デフォルト向け)
+auth = "plain[passwd=/etc/ocserv/ocpasswd]"
+# 証明書認証 (パスワード認証と排他。切り替える場合は上記をコメントアウト)
 #auth = "certificate"
 #auth = "radius[config=/etc/radiusclient/radiusclient.conf,groupconfig=true]"

@@ -124,8 +126,8 @@
 # certificate renewal (they are checked and reloaded periodically;
 # a SIGHUP signal to main server will force reload).

-#server-cert = /etc/ocserv/server-cert.pem
-#server-key = /etc/ocserv/server-key.pem
+server-cert = /etc/ocserv/server.crt
+server-key = /etc/ocserv/server.key

 # Diffie-Hellman parameters. Only needed if for old (pre 3.6.0
 # versions of GnuTLS for supporting DHE ciphersuites.
@@ -151,7 +153,7 @@
 # The Certificate Authority that will be used to verify
 # client certificates (public keys) if certificate authentication
 # is set.
-#ca-cert = /etc/ocserv/ca.pem
+ca-cert = /etc/ocserv/privateca.crt

 # The number of sub-processes to use for the security module (authentication)
 # processes. Typically this should not be set as the number of processes
@@ -475,7 +477,7 @@

 # The default domain to be advertised. Multiple domains (functional on
 # openconnect clients) can be provided in a space separated list.
-default-domain = example.com
+default-domain = vpn.local
 #default-domain = "example.com one.example.com"

 # The pool of addresses that leases will be given from. If the leases
@@ -486,7 +488,7 @@
 # Note that, you could use addresses from a subnet of your LAN network if you
 # enable [proxy arp in the LAN interface](http://ocserv.gitlab.io/www/recipes-ocserv-pseudo-bridge.html);
 # in that case it is recommended to set ping-leases to true.
-ipv4-network = 192.168.1.0
+ipv4-network = 10.255.255.0
 ipv4-netmask = 255.255.255.0

 # An alternative way of specifying the network:
@@ -509,7 +511,8 @@
 # The advertized DNS server. Use multiple lines for
 # multiple servers.
 # dns = fc00::4be0
-dns = 192.168.1.1
+dns = 8.8.8.8
+dns = 8.8.4.4

 # The NBNS server (if any)
 #nbns = 192.168.1.3
```

### IP フォワード有効化

VPN クライアントのパケットをサーバ側でルーティングするために有効化します。

```bash
echo "net.ipv4.ip_forward = 1" | \
    sudo tee -a /etc/sysctl.d/99-ocserv.conf
sudo sysctl -p /etc/sysctl.d/99-ocserv.conf
```

### NAT 設定 (iptables)

NAT 設定は iptables コマンドを使用します。  
また、再起動後も設定を維持する場合は `iptables-persistent` を使用します。

```bash
sudo apt-get install -y iptables-persistent
```

VPN クライアントのトラフィックをサーバの物理 NIC 経由で転送する場合に設定します。  
NIC 名 (`ens32`) は環境に合わせて変更してください。

```bash
# NIC 名の確認
ip link show

# NAT 設定 (eth0 は環境に合わせて変更)
sudo iptables -t nat -A POSTROUTING -s 10.255.255.0/24 -o ens32 -j MASQUERADE
sudo iptables -A FORWARD -s 10.255.255.0/24 -j ACCEPT
sudo iptables -A FORWARD -d 10.255.255.0/24 -j ACCEPT
```

```bash
sudo apt-get install -y iptables-persistent
sudo netfilter-persistent save
```

### ファイアウォール設定 (ufw を使用する場合)

```bash
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw reload
```

### ocserv サービス起動

```bash
sudo systemctl enable --now ocserv
sudo systemctl status ocserv
```


---

## クライアント設定

### インストール

```bash
sudo apt-get update
sudo apt-get install -y openconnect
```

### CA 証明書の配置

サーバから `privateca.crt` をクライアントへ安全にコピーします。

```bash
scp ubuntu@192.168.222.145:$HOME/ocserv-certs/privateca.crt .
```

### VPN 接続 (パスワード認証)

```bash
sudo openconnect \
    --cafile $HOME/privateca.crt \
    https://192.168.222.145:443
```

接続後、ユーザ名とパスワードの入力を求められます。

```text
POST https://192.168.222.145/
Connected to 192.168.222.145:443
SSL negotiation with 192.168.222.145
Connected to HTTPS on 192.168.222.145 with ciphersuite (TLS1.2)-(ECDHE-SECP256R1)-(RSA-SHA256)-(AES-256-GCM)
XML POST enabled
Please enter your username.
Username:vpn-user01
POST https://192.168.222.145/auth
Please enter your password.
Password:
```

### VPN 接続 (証明書認証)

サーバから `client.p12` をクライアントへコピーし、接続します。

```bash
# クライアント証明書の取得
scp ubuntu@192.168.222.145:$HOME/ocserv-certs/client.p12 .

# 証明書認証で接続
sudo openconnect \
    --cafile $HOME/privateca.crt \
    --certificate $HOME/client.p12 \
    https://192.168.222.145:443
```

PKCS#12 ファイルにパスワードを設定した場合は、パスワード入力を求められます。

```text
POST https://192.168.222.145/
Connected to 192.168.222.145:443
Enter PKCS#12 pass phrase:
Using client certificate 'vpn-user01'
SSL negotiation with 192.168.222.145
Connected to HTTPS on 192.168.222.145 with ciphersuite (TLS1.2)-(ECDHE-SECP256R1)-(RSA-SHA256)-(AES-256-GCM)
XML POST enabled
Please enter your username.
Username:vpn-user01
POST https://192.168.222.145/auth
SSL negotiation with 192.168.222.145
Connected to HTTPS on 192.168.222.145 with ciphersuite (TLS1.2)-(ECDHE-SECP256R1)-(RSA-SHA256)-(AES-256-GCM)
Please enter your password.
Password:
```

### バックグラウンド接続

```bash
sudo openconnect \
    --cafile $HOME/privateca.crt \
    --background \
    --pid-file /var/run/openconnect.pid \
    https://192.168.222.145:443
```

### VPN 切断

```bash
sudo kill $(cat /var/run/openconnect.pid)
```

---

## 動作確認

### サーバ側ログ確認

```bash
sudo journalctl -u ocserv -f
```

### クライアント側: 接続確認

VPN 接続後にインタフェースとルーティングを確認します。

```bash
# VPN インタフェース確認 (vpn0 または tun0 が追加されていること)
ip addr show

# ルーティング確認
ip route

# VPN サーバ (ゲートウェイ) への疎通確認
ping 10.255.255.1
```

### 接続中ユーザの確認 (サーバ側)

```bash
sudo occtl show users
```

```text
  id     user    vhost             ip         vpn-ip device   since    dtls-cipher    status
2681 vpn-user01  default 192.168.222.146 10.255.255.108  vpns0  1m:30s  (AES-256-GCM) connected
```

---

## トラブルシューティング

- **ocserv が起動しない**

    設定ファイルの構文エラーを確認します。

    ```bash
    sudo ocserv --config /etc/ocserv/ocserv.conf --foreground
    ```

- **証明書エラーが発生する**

    サーバ証明書の SAN に接続先 IP または DNS 名が含まれているか確認します。

    ```bash
    openssl x509 -in /etc/ocserv/server.crt -text -noout | grep -A1 "Subject Alternative"
    ```

- **DTLS が使用されない場合**

    UDP 443 がファイアウォールでブロックされている可能性があります。TLS のみで接続テストする場合は `--no-dtls` を使用します。

    ```bash
    sudo openconnect --no-dtls --cafile $HOME/privateca.crt https://192.168.222.145:443
    ```
