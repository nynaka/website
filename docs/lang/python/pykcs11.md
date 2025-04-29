[PyKCS11](https://github.com/LudovicRousseau/PyKCS11/)
===

[PyKCS11](https://github.com/LudovicRousseau/PyKCS11/) は、Python で PKCS#11 インタフェース関数を利用するラッパーライブラリです。  
他には python-pkcs11 というライブラリもあります。

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

[Welcome to PyKCS11’s documentation — PyKCS11 1.5.17 documentation](https://pkcs11wrap.sourceforge.io/api/) に API ドキュメントはあるのですが、関数単位の解説を読んでも、ある程度使い方を把握した人じゃないとわからない気がしました。  

ドキュメントの最後の方にある [PyKCS11 samples codes](https://pkcs11wrap.sourceforge.io/api/samples.html) や [テストコード](https://github.com/LudovicRousseau/PyKCS11/tree/master/test) に動作するコードがあるので、それをベースにカスタマイズした方が要領が得られると思います。


<details open="">
<summary>対称鍵で暗号・復号</summary>

```python
import binascii

import PyKCS11

""" 設定 """
# PKCS11 モジュールのパス
PKCS11_LIB = "/usr/lib/softhsm/libsofthsm2.so"

# User PIM
PIN = "1234"

# 暗号化対称データ
plaintext = b"plaintext data" * 4
""" 設定 """


def encrypt(
    session: PyKCS11.Session,
    key_handle: PyKCS11.CK_OBJECT_HANDLE,
    mechanism: str,
    plaintext: bytes,
    verbose: bool = True,
) -> dict:
    if mechanism == "AES_CBC_PAD":
        # 16バイトのランダムな初期化ベクタを生成
        iv = session.generateRandom(16)
        # AES CBC パディング
        enc_mechanism = PyKCS11.Mechanism(PyKCS11.CKM_AES_CBC_PAD, iv)
        # 暗号化
        ciphertext = session.encrypt(key_handle, plaintext, enc_mechanism)
        return_data = {"ciphertext": ciphertext, "iv": iv}

    elif mechanism == "AES_CTR":
        CTR_BITS = 128
        # CTR の初期化ベクタを生成
        iv = session.generateRandom(CTR_BITS // 8)
        # AES CTR
        enc_mechanism = PyKCS11.AES_CTR_Mechanism(CTR_BITS, iv)
        # 暗号化
        ciphertext = session.encrypt(key_handle, plaintext, enc_mechanism)
        return_data = {"ciphertext": ciphertext, "ctr_bits": CTR_BITS, "iv": iv}

    elif mechanism == "AES_GCM":
        # GCM の初期化ベクタを生成
        iv = session.generateRandom(12)
        aad = session.generateRandom(16)
        tagBits = 128
        # AES GCM
        enc_mechanism = PyKCS11.AES_GCM_Mechanism(iv, aad, tagBits)
        # 暗号化
        ciphertext = session.encrypt(key_handle, plaintext, enc_mechanism)
        return_data = {
            "ciphertext": ciphertext,
            "iv": iv,
            "aad": aad,
            "tagBits": tagBits,
        }

    if verbose:
        print("return_data:", return_data)

    return return_data


def decrypt(
    session: PyKCS11.Session,
    key_handle: PyKCS11.CK_OBJECT_HANDLE,
    mechanism: str,
    encrypted_data: dict,
    verbose: bool = True,
):
    if mechanism == "AES_CBC_PAD":
        iv = encrypted_data["iv"]
        ciphertext = encrypted_data["ciphertext"]

        # AES CBC パディング
        enc_mechanism = PyKCS11.Mechanism(PyKCS11.CKM_AES_CBC_PAD, iv)
        # 復号
        plaintext_data_array = session.decrypt(key_handle, ciphertext, enc_mechanism)

    elif mechanism == "AES_CTR":
        iv = encrypted_data["iv"]
        ctr_bits = encrypted_data["ctr_bits"]
        ciphertext = encrypted_data["ciphertext"]
        # AES CTR
        enc_mechanism = PyKCS11.AES_CTR_Mechanism(ctr_bits, iv)
        # 復号
        plaintext_data_array = session.decrypt(key_handle, ciphertext, enc_mechanism)

    elif mechanism == "AES_GCM":
        iv = encrypted_data["iv"]
        aad = encrypted_data["aad"]
        tagBits = encrypted_data["tagBits"]
        ciphertext = encrypted_data["ciphertext"]
        # AES GCM
        enc_mechanism = PyKCS11.AES_GCM_Mechanism(iv, aad, tagBits)
        # 復号
        plaintext_data_array = session.decrypt(key_handle, ciphertext, enc_mechanism)

    if verbose:
        print("return_data:", plaintext_data_array)

    return plaintext_data_array


if __name__ == "__main__":
    pkcs11 = PyKCS11.PyKCS11Lib()
    # PKCS11 ライブラリのロード
    pkcs11.load(PKCS11_LIB)

    # スロットを取得し、セッションを開始
    slot = pkcs11.getSlotList(tokenPresent=True)[0]
    session = pkcs11.openSession(
        slot, PyKCS11.CKF_RW_SESSION | PyKCS11.CKF_SERIAL_SESSION
    )

    # ログイン
    session.login(PIN)

    # AES 256 キーを作成
    key_template = [
        (PyKCS11.CKA_CLASS, PyKCS11.CKO_SECRET_KEY),
        (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_AES),
        (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),
        (PyKCS11.CKA_LABEL, "aes-key"),
        (PyKCS11.CKA_ID, binascii.unhexlify("9876")),
        (PyKCS11.CKA_VALUE_LEN, 32),  # 32バイト = 256ビット
        (PyKCS11.CKA_PRIVATE, PyKCS11.CK_FALSE),
        (PyKCS11.CKA_ENCRYPT, PyKCS11.CK_TRUE),
        (PyKCS11.CKA_DECRYPT, PyKCS11.CK_TRUE),
        (PyKCS11.CKA_SIGN, PyKCS11.CK_FALSE),
        (PyKCS11.CKA_VERIFY, PyKCS11.CK_FALSE),
    ]
    # AES-256 対称鍵の生成
    key_handle = session.generateKey(key_template)

    """ AES_CBC_PAD """
    encrypted_data = encrypt(
        session, key_handle, "AES_CBC_PAD", plaintext, verbose=False
    )
    decrypted_data_array = decrypt(
        session, key_handle, "AES_CBC_PAD", encrypted_data, verbose=False
    )
    decrypted_data = bytes(decrypted_data_array)
    print("AES_CBC_PAD decrypt: ", decrypted_data)
    # 暗号化・復号の検証
    assert plaintext == decrypted_data

    """ AES_CTR """
    encrypted_data = encrypt(session, key_handle, "AES_CTR", plaintext, verbose=False)
    decrypted_data_array = decrypt(
        session, key_handle, "AES_CTR", encrypted_data, verbose=False
    )
    decrypted_data = bytes(decrypted_data_array)
    print("AES_CTR decrypt: ", decrypted_data)
    # 暗号化・復号の検証
    assert plaintext == decrypted_data

    """ AES_GCM """
    encrypted_data = encrypt(session, key_handle, "AES_GCM", plaintext, verbose=False)
    decrypted_data_array = decrypt(
        session, key_handle, "AES_GCM", encrypted_data, verbose=False
    )
    decrypted_data = bytes(decrypted_data_array)
    print("AES_GCM decrypt: ", decrypted_data)
    # 暗号化・復号の検証
    assert plaintext == decrypted_data

    # 鍵削除
    session.destroyObject(key_handle)

    # セッションを閉じる
    session.logout()
    session.closeSession()
```

</details>

<details open="">
<summary>暗号化鍵の Wrap/UnWrap</summary>

```python
import binascii
import secrets

from PyKCS11 import PyKCS11

""" 設定 """
# PKCS11 モジュールのパス
PKCS11_LIB = "/usr/lib/softhsm/libsofthsm2.so"

# User PIN
PIN = "1234"

# 暗号化対称データ
plaintext = b"plaintext data" * 4
""" 設定 """


def encrypt_decrypt(
    key_handle: PyKCS11.LowLevel.CK_OBJECT_HANDLE, plaintext: bytes
) -> None:
    """暗号・復号テスト

    指定された鍵ハンドルを使用して、指定した平文を暗号化⇒復号できることを確認する。

    Args:
        key_handle (PyKCS11.LowLevel.CK_OBJECT_HANDLE): 暗号化に使用する鍵ハンドル
        plaintext (bytes): 暗号化する平文

    Returns: None
    """
    # イニシャルベクタ (16バイト)
    iv = secrets.token_bytes(16)

    # 暗号化
    enc_mechanism = PyKCS11.Mechanism(PyKCS11.CKM_AES_CBC_PAD, iv)
    ciphertext = session.encrypt(key_handle, plaintext, enc_mechanism)
    print("Encrypted:", binascii.hexlify(bytearray(ciphertext)))

    # 復号
    dec_mechanism = PyKCS11.Mechanism(PyKCS11.CKM_AES_CBC_PAD, iv)
    decrypted = session.decrypt(key_handle, ciphertext, dec_mechanism)
    print("Decrypted:", decrypted)
    assert plaintext == bytes(decrypted)


# PKCS11 モジュールをロード
pkcs11 = PyKCS11.PyKCS11Lib()
pkcs11.load(PKCS11_LIB)

# スロットを取得し、セッションを開始
slot = pkcs11.getSlotList(tokenPresent=True)[0]
session = pkcs11.openSession(slot, PyKCS11.CKF_RW_SESSION | PyKCS11.CKF_SERIAL_SESSION)

# ログイン
session.login(PIN)

# AES 256 キーを作成
key_template = [
    (PyKCS11.CKA_LABEL, "aes-key"),
    (PyKCS11.CKA_ID, binascii.unhexlify("9876")),
    (PyKCS11.CKA_VALUE_LEN, 32),  # 32バイト = 256ビット
    (PyKCS11.CKA_ENCRYPT, PyKCS11.CK_TRUE),
    (PyKCS11.CKA_DECRYPT, PyKCS11.CK_TRUE),
    (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),  # トークンに保存
    (PyKCS11.CKA_PRIVATE, PyKCS11.CK_FALSE),
    (PyKCS11.CKA_EXTRACTABLE, PyKCS11.CK_TRUE),
]
# AES-256 対称鍵の生成
enckey_handle = session.generateKey(key_template)
wrapkey_handle = session.generateKey(key_template)

# 暗号鍵のエクスポート
# wrap using CKM_AES_KEY_WRAP
wrap_mechanism = PyKCS11.Mechanism(PyKCS11.CKM_AES_KEY_WRAP)
wrapped_key = session.wrapKey(wrapkey_handle, enckey_handle, wrap_mechanism)
assert wrapped_key is not None

# 暗号化鍵を削除
session.destroyObject(enckey_handle)

# unwrap
template = [
    (PyKCS11.CKA_CLASS, PyKCS11.CKO_SECRET_KEY),
    (PyKCS11.CKA_KEY_TYPE, PyKCS11.CKK_AES),
    (PyKCS11.CKA_TOKEN, PyKCS11.CK_TRUE),
    (PyKCS11.CKA_PRIVATE, PyKCS11.CK_FALSE),
    (PyKCS11.CKA_ENCRYPT, PyKCS11.CK_TRUE),
    (PyKCS11.CKA_DECRYPT, PyKCS11.CK_TRUE),
    (PyKCS11.CKA_SIGN, PyKCS11.CK_FALSE),
    (PyKCS11.CKA_VERIFY, PyKCS11.CK_FALSE),
]
unwrappedkey_handle = session.unwrapKey(
    wrapkey_handle, wrapped_key, template, wrap_mechanism
)
assert unwrappedkey_handle is not None

# 暗号・復号テスト
encrypt_decrypt(unwrappedkey_handle, plaintext)

# 鍵削除
session.destroyObject(unwrappedkey_handle)
session.destroyObject(wrapkey_handle)

# セッションを閉じる
session.logout()
session.closeSession()
```

</details>
