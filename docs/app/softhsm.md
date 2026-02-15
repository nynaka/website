[SoftHSM](https://github.com/softhsm/SoftHSMv2)
===

## インストール

- Debian 系 Linux (Debian Linux、Ubuntu Linux 他)

    ```bash
    sudo apt install -y softhsm2 opensc
    ```

- Redhat 系 Linux (Fedora、Alma Linux、Rocky Linux)

    ```bash
    sudo dnf install -y softhsm opensc
    ```

- ソースからビルドする場合

    - ビルドツールのインストール

        ```bash
        sudo apt install -y build-essential autoconf libtool libssl-dev
        ```

    - ソースからビルドする

        ```bash
        git clone https://github.com/softhsm/SoftHSMv2
        cd SoftHSMv2
        ./autogen.sh
        ./configure --enable-ecc --enable-eddsa
        make && sudo make install
        ```

### PKCS11 ライブラリの位置

| OS                                                  | ライブラリパス                        |
| :-------------------------------------------------- | :------------------------------------ |
| Debian 系 Linux (Debian Linux、Ubuntu Linux 他)     | /usr/lib/softhsm/libsofthsm2.so       |
| Redhat 系 Linux (Fedora、Alma Linux、RockyLinux 他) | /usr/lib64/softhsm/libsofthsm.so      |
| ソースからビルドした場合                            | /usr/local/lib/softhsm/libsofthsm2.so |
|                                                     |                                       |


## softhsm2-util

`softhsm2-util` は SoftHSM トークンを初期化および管理するためのコマンドラインツールです。

- トークン情報の表示

    ```bash
    softhsm2-util --show-slots
    ```

- トークンの初期化

    ```bash
    # softhsm2-util --init-token --slot <slot> --label <label>
    softhsm2-util --init-token --slot 0 --label "SoftHSM Token"
    ```

    SO (Security Officer) PIN と user PIN の設定を求められます。

    > Please enter the new SO PIN:
    > Please re-enter the new SO PIN:
    > Please enter the new user PIN:
    > Please re-enter the new user PIN:

- 鍵のインポート

    ```bash
    # softhsm2-util --import <path> --slot <slot> --label <label> --id <hex-id>
    softhsm2-util --import mykey.pem --slot 0 --label "mykey" --id 1234
    ```

## PKCS11 での操作

- PKCS11 ライブラリパスの設定

    ```bash
    export PKCS11_MODULE_PATH="/usr/lib64/softhsm/libsofthsm.so"
    ```

- サポートアルゴリズムの確認

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH -M
    ```

- トークンの初期化

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH \
        --init-token \
        --slot 0 \
        --so-pin 1234 \
        --label "SoftHSM Token"
    ```

- User PIN の設定

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH \
        --init-pin \
        --so-pin 1234 \
        --pin 1234
    ```

- スロット番号の確認

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH -L
    ```

- 鍵作成

    - 対称鍵

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keygen \
            --key-type AES:32 \
            --label "aeskey" \
            --login --pin 1234
        ```

    - RSA キーペア

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keypairgen \
            --key-type rsa:4096 \
            --label "rsakey" \
            --login --pin 1234
        ```

    - ECDSA キーペア

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keypairgen \
            --key-type EC:prime256v1 \
            --label "ecdsakey" \
            --login --pin 1234
        ```

    - EdDSA(ed25519) キーペア

        ```bash
        pkcs11-tool --module $PKCS11_MODULE_PATH \
            --keypairgen \
            --key-type EC:edwards25519 \
            --label "ed25519key" \
            --login --pin 1234
        ```

        !!! note
            **--key-type** には **curve25519** も指定できますが、こちらは ECDH の X25519 になります。

- 登録鍵一覧の取得

    ```bash
    pkcs11-tool --module $PKCS11_MODULE_PATH -O \
        --login \
        --pin 1234
    ```


## OpenSSL コマンド化利用する

!!! note
    openssl 3.0.x のように古い OpenSSL だと、謎のコアダンプに悩まされることがあるようです。  
    Ubuntu 24.04 で試してみたら、Github Copilot でも解決できなかった。

### PKCS11 Engine (OpenSSL3 から非推奨)

#### インストール

- Debian 系 Linux (Debian Linux、Ubuntu Linux 他)

    ```bash
    sudo apt install -y libengine-pkcs11-openssl
    ```

- Redhat 系 Linux (Fedora、Alma Linux、Rocky Linux)

    ```bash
    sudo dnf install -y openssl-pkcs11
    ```

#### 使い方

- PKCS11 ライブラリパスの設定

    ```bash
    export PKCS11_MODULE_PATH="/usr/local/lib/softhsm/libsofthsm2.so"
    ```

- 自己署名証明書の作成

    !!! note
        自己証明書を 自己 CA 証明書として使用することを想定しています。

    ```bash
    openssl req -new -x509 -days 3650 \
        -engine pkcs11 \
        -keyform engine \
        -key "pkcs11:token=SoftHSM Token;object=rsakey;type=private;pin-value=1234" \
        -out selfsigned.crt \
        -subj "/CN=example.com" \
        -addext "subjectAltName=DNS:example.com,DNS:localhost,IP:127.0.0.1"
    ```

- CSR (Certificate Signing Request：証明書署名要求) の作成

    ```bash
    openssl req -new \
        -engine pkcs11 \
        -keyform engine \
        -key "pkcs11:token=SoftHSM Token;object=user-rsakey;type=private;pin-value=1234" \
        -out request.csr \
        -subj "/CN=host1.example.com" \
        -addext "subjectAltName=DNS:host1.example.com,DNS:localhost,IP:127.0.0.1"
    ```

- CRT (Certificate: デジタル証明書) の作成 (CSR に署名を付ける)

    ```bash
    # SoftHSM内のCA鍵を使って、request.csr に署名し user.crt を作る
    openssl x509 -req -days 365 \
        -in request.csr \
        -CA selfsigned.crt \
        -CAkey "pkcs11:token=SoftHSM Token;object=rsakey;type=private;pin-value=1234" \
        -engine pkcs11 \
        -CAkeyform engine \
        -CAcreateserial \
        -out user.crt
    ```

### PKCS11 Provider (OpenSSL3 から推奨)

#### インストール

```bash
sudo apt install -y pkcs11-provider
```



## サンプルコード

### PyKCS11

- 非対称鍵作成

    ```python
    from asn1crypto.keys import ECDomainParameters, NamedCurve
    from PyKCS11 import PyKCS11

    """ 設定 """
    # PKCS11 モジュールのパス
    PKCS11_MODULE_PATH = "/usr/local/lib/softhsm/libsofthsm2.so"

    # User PIN
    PIN = "1234"
    """ 設定 """


    if __name__ == "__main__":
        # PKCS11 モジュールを初期化
        pkcs11 = PyKCS11.PyKCS11Lib()
        pkcs11.load(PKCS11_MODULE_PATH)

        # スロットを取得
        slots = pkcs11.getSlotList()
        slot = slots[0]

        # スロットにログイン
        session = pkcs11.openSession(
            slot, PyKCS11.CKF_SERIAL_SESSION | PyKCS11.CKF_RW_SESSION
        )
        session.login(PIN)

        # Select the curve to be used for the keys
        # curve = "secp256r1"    # ECDSA SECP256R1
        curve = "secp384r1"  # ECDSA SECP384R1

        # Setup the domain parameters, unicode conversion needed
        # for the curve string
        domain_params = ECDomainParameters(name="named", value=NamedCurve(curve))
        ec_params = domain_params.dump()

        keyID = (0x22,)
        label = "test"

        ec_public_tmpl = [
            (PyKCS11.CKA_CLASS, PyKCS11.CKO_PUBLIC_KEY),
            (PyKCS11.CKA_PRIVATE, PyKCS11.CK_FALSE),
            (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_ENCRYPT, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_VERIFY, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_WRAP, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_ECDSA),
            (PyKCS11.CKA_EC_PARAMS, ec_params),
            (PyKCS11.CKA_LABEL, label),
            (PyKCS11.CKA_ID, keyID),
        ]
        ec_priv_tmpl = [
            (PyKCS11.CKA_CLASS, PyKCS11.CKO_PRIVATE_KEY),
            (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_ECDSA),
            (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_SENSITIVE, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_DECRYPT, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_SIGN, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_UNWRAP, PyKCS11.CK_TRUE),
            (PyKCS11.CKA_LABEL, label),
            (PyKCS11.CKA_ID, keyID),
        ]

        (pubKey, privKey) = session.generateKeyPair(
            ec_public_tmpl, ec_priv_tmpl, mecha=PyKCS11.MechanismECGENERATEKEYPAIR
        )
        # session.destroyObject(pubKey)
        # session.destroyObject(privKey)

        public_key_label = "MyEd25519PublicKey"
        private_key_label = "MyEd25519PrivateKey"
        ED25519_OID_DER = b"\x06\x03\x2b\x65\x70"  # OID 1.3.101.112
        mechanism = PyKCS11.Mechanism(PyKCS11.CKM_EC_EDWARDS_KEY_PAIR_GEN, None)

        # 公開鍵のテンプレート
        # CKK_EC_EDWARDS は EdDSA キーを示す標準的なキータイプ (PKCS#11 v3.0)
        # CKA_EC_PARAMS に Ed25519 の OID を指定
        pub_template = [
            (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_EC_EDWARDS),
            (PyKCS11.CKA_LABEL, public_key_label),
            (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),  # トークン上に永続的に保存
            (PyKCS11.CKA_VERIFY, PyKCS11.CK_TRUE),  # 署名検証に使用可能
            (PyKCS11.CKA_EC_PARAMS, ED25519_OID_DER),  # 曲線パラメータ (Ed25519 OID)
            # (PyKCS11.CKA_ID, b'unique_id_pub'), # 必要ならID設定
        ]
        # 秘密鍵のテンプレート
        priv_template = [
            (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_EC_EDWARDS),
            (PyKCS11.CKA_LABEL, private_key_label),
            (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),  # トークン上に永続的に保存
            (PyKCS11.CKA_SIGN, PyKCS11.CK_TRUE),  # 署名生成に使用可能
            (PyKCS11.CKA_SENSITIVE, PyKCS11.CK_TRUE),  # 機密属性 (通常True)
            (PyKCS11.CKA_EXTRACTABLE, PyKCS11.CK_FALSE),  # 鍵のエクスポート不可 (通常False)
            # (PyKCS11.CKA_ID, b'unique_id_priv'), # 必要ならID設定
        ]

        pubKey, privKey = session.generateKeyPair(
            pub_template, priv_template, mecha=mechanism
        )
        # session.destroyObject(pubKey)
        # session.destroyObject(privKey)

        # セッションをログアウトして閉じる
        session.logout()
        session.closeSession()
    ```
