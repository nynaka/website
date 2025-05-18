PKCS11 (C言語)
===

!!! important
    SoftHSM が前提です。他の HSM 製品の場合は、製品ごとの対応が必要になる可能性があります。

## SoftHSM のインストール

- Redhat系 (Fedora、Alma Linux9、Rocky Linux9)

    ```bash
    sudo dnf install -y softhsm opensc
    ```

- Debian系 (Debian Linux 12、Ubuntu Linux 22.04)

    ```bash
    sudo apt install -y softhsm2 opensc
    ```

## pkcs11-tool での SoftHSM の操作

- ライブラリパスの設定

    ライブラリパスを環境変数に設定しておくと少しだけ便利です。

    ```bash
    export LIBPATH=/usr/lib/softhsm/libsofthsm2.so
    ```

- サポートアルゴリズムの確認

    ```bash
    sudo pkcs11-tool --module $LIBPATH -M
    ```

- トークンの初期化

    ```bash
    sudo pkcs11-tool --module $LIBPATH \
        --init-token \
        --label "init-token" \
        --so-pin 1234
    ```

- user pin の設定

    ```bash
    sudo pkcs11-tool --module $LIBPATH \
        --init-pin \
        --so-pin 1234 \
        --pin 4321
    ```

- スロットの確認

    ```bash
    sudo pkcs11-tool --module $LIBPATH -L
    ```

- 登録鍵一覧取得

    ```bash
    pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -O \
        --login --pin 4321
    ```

- 鍵の作成

    - 対象鍵の作成

        ```bash
        sudo pkcs11-tool --module $LIBPATH \
            --keygen \
            --key-type AES:32 \
            --label "aeskey" \
            --login --pin 4321
        ```

    - RSA 非対称鍵の作成

        ```bash
        sudo pkcs11-tool --module $LIBPATH \
            --keypairgen \
            --key-type rsa:2048 \
            --label "rsakey" \
            --login --pin 4321
        ```

    - EC 非対称鍵 (NIST P-256 の例) の作成

        ```bash
        sudo pkcs11-tool --module $LIBPATH \
            --keypairgen \
            --key-type EC:secp256r1 \
            --label "ecdsakey" \
            --login --pin 4321
        ```

- 鍵の削除
    - 対称鍵
        - ID 指定削除

            ```bash
            pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
                --login --pin 1234 \
                --delete -d 01 -y secrkey
            ```

        - Label 指定削除

            ```bash
            pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
                --login --pin 1234 \
                --delete -y secrkey --label "aeskey"
            ```


## サンプルコード

### 対称鍵作成

### RSA キーペア作成

### 登録鍵一覧取得

### 鍵削除
