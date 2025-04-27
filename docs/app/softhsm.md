[SoftHSM](https://github.com/softhsm/SoftHSMv2)
===

## インストール

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

## PKCS11 での操作

- サポートアルゴリズムの確認

    ```bash
    pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -M
    ```

- トークンの初期化

    ```bash
    pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so \
        --init-token \
        --slot 0 \
        --so-pin 1234 \
        --label "SoftHSM Token"
    ```

- User PIN の設定

    ```bash
    pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so \
        --init-pin \
        --so-pin 1234 \
        --pin 1234
    ```

- スロット番号の確認

    ```bash
    pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -L
    ```

- 登録鍵一覧の取得

    ```bash
    pkcs11-tool --module /usr/local/lib/softhsm/libsofthsm2.so -O \
        --login \
        --pin 1234
    ```


## サンプルコード

### PyKCS11

- 非対称鍵作成

    ```python
    from asn1crypto.keys import ECDomainParameters, NamedCurve
    from PyKCS11 import PyKCS11

    """ 設定 """
    # PKCS11 モジュールのパス
    PKCS11_LIB = "/usr/local/lib/softhsm/libsofthsm2.so"

    # User PIN
    PIN = "1234"
    """ 設定 """


    if __name__ == "__main__":
        # PKCS11 モジュールを初期化
        pkcs11 = PyKCS11.PyKCS11Lib()
        pkcs11.load(PKCS11_LIB)

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
