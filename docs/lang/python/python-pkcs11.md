[python-pkcs11](https://github.com/pyauth/python-pkcs11)
===

[python-pkcs11](https://github.com/pyauth/python-pkcs11) は、Python で PKCS#11 インタフェース関数を利用するラッパーライブラリです。  
他には pykcs11 というライブラリもあります。

## 実行環境準備

### SoftHSM のインストール

- Debian Linux

    ```bash
    sudo apt install -y softhsm2 libsofthsm2 opensc
    ```

### SoftHSM の初期化

- 初期化

    ```bash
    softhsm2-util \
        --init-token \
        --slot 0 \
        --label "SoftHSMToken" \
        --so-pin 4321 \
        --pin 1234
    ```

- トークンの確認

    ```bash
    softhsm2-util --show-slots
    ```

    ```bash
    pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -L
    ```

<details>
<summary>pkcs11-tool での SoftHSM 操作</summary>

- サポートアルゴリズム一覧の確認

    ```bash
    pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -M
    ```

- 登録鍵一覧取得

    ```bash
    pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so -O \
        --login --pin 1234
    ```

- 鍵作成

    - 対称鍵

        ```bash
        pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
            --login --pin 1234 \
            --keygen \
            --key-type aes:32 \
            --id 01 \
            --label "aeskey"
        ```

    - RSA 鍵

        ```bash
        pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
            --login --pin 1234 \
            --keypairgen \
            --key-type rsa:4096 \
            --id 02 \
            --label "rsakey"
        ```

        公開鍵のエクスポート

        ```bash
        pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
            --login --pin 1234 \
            --read-object \
            --type pubkey \
            --id 02 \
            --output-file public_key.der
        ```

        DER 形式 ⇒ PEM 形式に変換

        ```bash
        openssl rsa \
            -in public_key.der \
            -inform der -pubin \
            -outform pem \
            -out public_key.pem
        ```

    - EC 鍵 (NIST P-256 の例)

        ```bash
        pkcs11-tool --module /usr/lib/softhsm/libsofthsm2.so \
            --login --pin 1234 \
            --keypairgen \
            --key-type EC:secp256r1 \
            --id 03 \
            --label "eckey"
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
</details>


## サンプルコード

[Python PKCS#11 - High Level Wrapper API — Python PKCS#11  documentation](https://python-pkcs11.readthedocs.io/en/latest/) に API ドキュメントはあるのですが、ある程度使い方を把握した人じゃないとわからない気がしました。  
[Nitrokey HSMを想定した Getting Started](https://python-pkcs11.readthedocs.io/en/latest/opensc.html#getting-started) も用意されていますが、[python-pkcs11/tests](https://github.com/pyauth/python-pkcs11/tree/master/tests) に動作するコードがあるので、それをベースにカスタマイズした方が要領が得られると思います。

<details open="">
<summary>対称鍵で暗号・復号</summary>

```python
import pkcs11

""" 設定 """
# PKCS11 モジュールのパス
PKCS11_LIB = "/usr/lib/softhsm/libsofthsm2.so"

# User PIN
PIN = "1234"

# 暗号化対称データ
plaintext = b"I hate working overtime..."
""" 設定 """

# Initialise our PKCS#11 library
lib = pkcs11.lib(PKCS11_LIB)
slots = lib.get_slots()
slot1, slot2 = slots
# print("Slot1: ", slot1)
# print("Slot2: ", slot2)

token = slot1.get_token()
with token.open(user_pin=PIN) as session:
    enckey = session.generate_key(
        pkcs11.KeyType.AES,
        256,
        template={
            pkcs11.Attribute.EXTRACTABLE: True,
            pkcs11.Attribute.SENSITIVE: False,
        },
    )

    iv = session.generate_random(128)
    crypttext = enckey.encrypt(plaintext, mechanism_param=iv)
    decrypttext = enckey.decrypt(crypttext, mechanism_param=iv)
    enckey.destroy()

    print("Plaintext:", plaintext)
    print("Encrypted:", crypttext)
    print("Decrypted:", decrypttext)
```

</details>


<details open="">
<summary>RSA 鍵で署名・署名検証</summary>

- サンプルコード

    ```python
    import hashlib
    import pkcs11
    from pkcs11.util.rsa import encode_rsa_public_key
    
    ### 設定 ###
    PKCS11_MODULE = "/usr/lib/softhsm/libsofthsm2.so"
    
    PIN = "1234"
    
    # 署名対象のメッセージ
    message = b"I hate working overtime..."
    ############
    
    
    def sign_message_body(
        message: bytes, private_key_handle: pkcs11.Key, public_key_handle: pkcs11.Key
    ) -> bytes:
        """
        指定された秘密鍵を使用してメッセージに署名し、メッセージと署名をファイルに書き込む。
    
        Args:
            message (bytes): 署名対象のメッセージ
            private_key_handle (pkcs11.Key): 署名に使用する秘密鍵のハンドル
            public_key_handle (pkcs11.Key): 署名検証に使用する公開鍵のハンドル
    
        Returns:
            bytes: 生成された署名
    
        Raises:
            pkcs11.exceptions.SignatureInvalid: 署名検証に失敗した場合
    
        Side Effects:
            メッセージを "message.txt" に書込む
            署名を "signature_body.bin" に書込む
        """
        with open("message.txt", "wb") as f:
            f.write(message)
    
        # 秘密鍵で署名 (SHA-256 と PKCS#1 v1.5 署名)
        signature = private_key_handle.sign(
            message, mechanism=pkcs11.Mechanism.SHA256_RSA_PKCS
        )
        # print("署名済み:", signature.hex())
        with open("signature_body.bin", "wb") as f:
            f.write(signature)
    
        # 署名の検証
        try:
            public_key_handle.verify(
                message, signature, mechanism=pkcs11.Mechanism.SHA256_RSA_PKCS
            )
            print("署名の検証に成功しました。")
        except pkcs11.exceptions.SignatureInvalid:
            print("署名の検証に失敗しました。")
    
        return signature
    
    
    def sign_message_hash(
        message: bytes, private_key_handle: pkcs11.Key, public_key_handle: pkcs11.Key
    ) -> bytes:
        """
        指定された秘密鍵を使用してメッセージのハッシュ値に署名し、
        メッセージハッシュ値と署名をファイルに書き込む。
    
        この関数は以下の手順を実行します:
        1. 入力メッセージのSHA-256ハッシュを計算する
        2. 計算されたハッシュにSHA-256プレフィックスを連結してDigestInfo構造を作成する
        3. DigestInfoをRSA PKCS#1 v1.5メカニズムを使用して署名する
        4. 対応する公開鍵を使用して署名を検証します。
    
        Args:
            message (bytes): 署名対象のメッセージ
            private_key_handle (pkcs11.Key): 署名に使用する秘密鍵のハンドル
            public_key_handle (pkcs11.Key): 署名検証に使用する公開鍵のハンドル
    
        Returns:
            bytes: 生成された署名
    
        Raises:
            pkcs11.exceptions.SignatureInvalid: 署名検証に失敗した場合
    
        Side Effects:
            メッセージを "message.txt" に書込む
            署名を "signature_body.bin" に書込む
        """
        message_hash = hashlib.sha256(message).digest()
        with open("message_hash.bin", "wb") as f:
            f.write(message_hash)
    
        # SHA-256用のDigestInfoのプレフィックス (DERエンコード済み)
        # DigestInfo構造は以下のようになっています:
        # SEQUENCE {
        #    SEQUENCE {
        #         OBJECT IDENTIFIER sha256 (2.16.840.1.101.3.4.2.1)
        #         NULL
        #    }
        #    OCTET STRING [ハッシュ値]
        # }
        # SHA-256の場合、プレフィックスは以下の16進数列になります。
        digest_info_prefix = bytes.fromhex("3031300d060960864801650304020105000420")
        # 上記プレフィックスとハッシュ値を連結してDigestInfoを作成
        digest_info = digest_info_prefix + message_hash
    
        # 署名対象は DigestInfo（ハッシュ値とアルゴリズム識別子の連結済みデータ）を使用
        # 署名には RSA PKCS#1 v1.5 の raw 署名機構を利用（ハッシュ計算は行われない）
        signature = private_key_handle.sign(
            digest_info, mechanism=pkcs11.Mechanism.RSA_PKCS
        )
        # print("署名済み:", signature.hex())
        with open("signature_hash.bin", "wb") as f:
            f.write(signature)
    
        # 署名の検証も同様に、DigestInfo を用いて行います
        try:
            public_key_handle.verify(
                digest_info, signature, mechanism=pkcs11.Mechanism.RSA_PKCS
            )
            print("署名の検証に成功しました。")
        except pkcs11.exceptions.SignatureInvalid:
            print("署名の検証に失敗しました。")
    
        return signature
    
    
    if __name__ == "__main__":
        # PKCS#11 モジュールのロード (SoftHSM2 の例)
        lib = pkcs11.lib(PKCS11_MODULE)
        slot1, slot2 = lib.get_slots()
    
        token = slot1.get_token()
        # トークンを開く
        with token.open(user_pin=PIN, rw=True) as session:
            # RSA キー生成
            public_key_handle, private_key_handle = session.generate_keypair(
                pkcs11.KeyType.RSA,
                2048,
                store=True,  # HSM に保存する
                public_template={
                    pkcs11.Attribute.ID: b"rsa-key",
                    pkcs11.Attribute.LABEL: "MyRSAKey",
                    pkcs11.Attribute.VERIFY: True,
                    pkcs11.Attribute.ENCRYPT: True,
                },
                private_template={
                    pkcs11.Attribute.ID: b"rsa-key",
                    pkcs11.Attribute.LABEL: "MyRSAKey",
                    pkcs11.Attribute.SIGN: True,
                    pkcs11.Attribute.DECRYPT: True,
                    pkcs11.Attribute.SENSITIVE: True,
                    pkcs11.Attribute.PRIVATE: True,
                },
            )
    
            # print(f"Public Key: {public_key_handle}")
            # print(f"Private Key: {private_key_handle}")
    
            # encode_rsa_public_key を使って DER 形式にエンコード
            der_encoded_public_key = encode_rsa_public_key(public_key_handle)
            # DER 形式でファイルに保存
            with open("public_key.der", "wb") as der_file:
                der_file.write(der_encoded_public_key)
    
            signature = sign_message_body(message, private_key_handle, public_key_handle)
            signature = sign_message_hash(message, private_key_handle, public_key_handle)
    
            public_key_handle.destroy()
            private_key_handle.destroy()
    ```

- openssl コマンドで検証する
    
    ```bash:公開鍵の DER ⇒ PEM 形式に変換
    openssl rsa -inform DER -in public_key.der -pubin -outform PEM -out public_key.pem
    ```

    - 検証

        ```bash:対象データそのものに対する署名の署名検証
        openssl dgst -sha256 \
            -verify public_key.pem \
            -signature signature_body.bin \
            message.txt
        ```
        
        ```bash:対象データのハッシュ値に対する署名の署名検証
        # 署名の期待値の生成
        openssl dgst -sha256 -binary message.txt > openssl_message_hash.bin
        (echo "3031300d060960864801650304020105000420" \
            | xxd -r -p; cat openssl_message_hash.bin) > expected_digestinfo.bin
        # 署名検証
        openssl pkeyutl -verify \
            -pubin \
            -inkey public_key.pem \
            -in expected_digestinfo.bin \
            -sigfile signature_hash.bin \
            -pkeyopt rsa_padding_mode:pkcs1
        ```

</details>
