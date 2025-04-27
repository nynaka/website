自己証明書作成手順
===

## 自己認証局署名鍵・証明書作成

```bash
openssl req \
    -new \
    -out ca.crt \
    -keyout ca.key \
    -x509 \
    -days 3650 \
    -newkey rsa:2048 \
    -nodes \
    -subj "/CN=PrivateCA"
```

## サーバ証明書作成

- CSR

    ```bash
    openssl req \
        -new \
        -out server.csr \
        -keyout server.key \
        -newkey rsa:2048 \
        -nodes \
        -subj "/CN=server.local"
    ```

- Subject Alternative Name の準備

    ここでは `san_server.txt` とします。

    ```text
    echo "subjectAltName = DNS:server.local, DNS:localhost, IP:192.168.1.1, IP:127.0.0.1" > san_server.txt
    ```

- CSR へ署名

    ```bash
    openssl x509 \
        -req \
        -days 3650 \
        -in server.csr \
        -out server.crt \
        -CA ca.crt \
        -CAkey ca.key \
        -CAcreateserial \
        -extfile san_server.txt
    ```

- 作成した証明書の確認

    ```bash
    openssl x509 -in server.crt -text -noout
    ```

- p12 形式ファイルの作成

    ```bash
    openssl pkcs12 -export \
        -in server.crt \
        -inkey server.key \
        -certfile ca.crt \
        -out server.p12 \
        -name "server_cert"
    ```

## クライアント証明書作成

- CSR

    ```bash
    openssl req \
        -new \
        -out client.csr \
        -keyout client.key \
        -newkey rsa:2048 \
        -nodes \
        -subj "/CN=client.local"
    ```

- Subject Alternative Name の準備

    ここでは `san_client.txt` とします。

    ```text
    echo "subjectAltName = DNS:client.local, IP:192.168.1.100" > san_client.txt
    ```

- CSR へ署名

    ```bash
    openssl x509 \
        -req \
        -days 3650 \
        -in client.csr \
        -out client.crt \
        -CA ca.crt \
        -CAkey ca.key \
        -CAcreateserial \
        -extfile san_client.txt
    ```

- 作成した証明書の確認

    ```bash
    openssl x509 -in client.crt -text -noout
    ```

- p12 形式ファイルの作成

    ```bash
    openssl pkcs12 -export \
        -in client.crt \
        -inkey client.key \
        -certfile ca.crt \
        -out client.p12 \
        -name "client_cert"
    ```

## 自己 CA を信頼する CA リストに追加する

- Ubuntu Linux

    ```bash
    sudo cp ca.crt /usr/local/share/ca-certificates/privateca.crt
    sudo update-ca-certificates
    ```

    この後、`/etc/ssl/certs/privateca.pem` が存在するはずです。

- Python (Requests ライブラリだけかもですが)

    Python では OS の信頼する CA リストは参照せず、`/usr/lib/python3/dist-packages/certifi/cacert.pem` または `venv/lib/python3.10/site-packages/certifi/cacert.pem` のような、Python環境の信頼する CA リストを参照しているようです。  
    このファイルに PrivateCA の証明書を追加すると requests が PrivateCA を信頼するようになります。
