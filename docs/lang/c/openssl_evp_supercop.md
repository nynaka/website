OpenSSL EVPインタフェースを使用した耐量子暗号アルゴリズム署名の性能測定
===

## 環境

| 項目         | 内容                |
| :----------- | :------------------ |
| OS           | Ubuntu Server 24.04 |
| OpenSSL      | 3.6.0               |
| liboqs       | 0.15.0              |
| oqs-provider | 0.10.0              |

## 必要なツールのインストール

```bash
sudo apt update
sudo apt install -y \
    build-essential cmake ninja-build \
    autoconf automake libtool git unzip
```

## 環境準備

- OpenSSL のインストール

    ```bash
    wget https://github.com/openssl/openssl/archive/refs/tags/openssl-3.6.0.tar.gz
    tar zxf openssl-3.6.0.tar.gz
    cd openssl-openssl-3.6.0/

    export CFLAGS="-march=native -O3"
    ./config enable-asm enable-ec_nistp_64_gcc_128
    make -j$(nproc)
    make install
    cd ..
    ```

- liboqs のインストール

    ```bash
    wget https://github.com/open-quantum-safe/liboqs/archive/refs/tags/0.15.0.tar.gz \
        -O liboqs-0.15.0.tar.gz \
        --no-check-certificate
    tar zxf liboqs-0.15.0.tar.gz
    cd liboqs-0.15.0
    mkdir build && cd build

    export CFLAGS="-O3 -march=native"
    cmake -GNinja \
        -DBUILD_SHARED_LIBS=ON \
        -DOQS_USE_OPENSSL=ON \
        -DBUILD_SHARED_LIBS=OFF \
        -DOQS_OPT_TARGET=x86_64_avx2 \
        ..
    ninja
    ninja install
    cd ../..
    ```

- oqs-provider のインストール

    ```bash
    wget https://github.com/open-quantum-safe/oqs-provider/archive/refs/tags/0.10.0.tar.gz \
        -O oqs-provider-0.10.0.tar.gz \
        --no-check-certificate
    tar zxf oqs-provider-0.10.0.tar.gz
    cd oqs-provider-0.10.0
    mkdir build && cd build
    cmake -GNinja \
        -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
        -DOPENSSL_ROOT_DIR=$INSTALL_DIR \
        -DCMAKE_PREFIX_PATH=$INSTALL_DIR \
        ..
    ninja
    ninja install
    cd ../..
    ```

- インストールされている OpenSSL がサポートしているアルゴリズムの確認

    ```bash
    export LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH
    openssl list -signature-algorithms
    ```

- [SUPERCOP](https://bench.cr.yp.to/supercop.html)

    ```bash
    wget https://bench.cr.yp.to/supercop/supercop-20251114.tar.xz
    tar Jxf supercop-20251114.tar.xz
    cd supercop-20251114
    ```

    - okcompilers/c

        **sed -i '2,$s/^/#/' okcompilers/c** のようにして、先頭の -O3 の行だけ残して、あとはコメント化する。

        ```bash
        gcc -march=native -mtune=native -O3 -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
        #gcc -march=native -mtune=native -Os -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
        #gcc -march=native -mtune=native -O2 -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
        #gcc -march=native -mtune=native -O -fwrapv -fPIC -fPIE -gdwarf-4 -Wall
        #clang -march=native -O3 -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
        #clang -march=native -Os -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
        #clang -march=native -O2 -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
        #clang -march=native -O -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
        #clang -mcpu=native -O3 -fwrapv -Qunused-arguments -fPIC -fPIE -gdwarf-4 -Wall
        ```

    - 初期化？

        あらかじめ実行しておかないとなぜかリンクエラーになるので、性能測定開始前の儀式と思って実行しています。

        ```bash
        ./do-part init
        ./do-part crypto_verify 32
        ./do-part crypto_hash sha512
        ./do-part crypto_stream chacha20
        ./do-part crypto_rng chacha20
        ./do-part crypto_sign ed25519
        ```

        たまにエラーが出たりしますが、深くがんが得ないで先に進みます。

    - SUPERCOP でビルドするための雛形準備

        - ソース格納ディレクトリの準備

            ソースコードは **crypto_sign/[アルゴリズム名]/[実装名]/** のディレクトリを作成し、ソースコードはその下の api.h と sign.c に記述するルールになっています。

            ```bash
            mkdir -p crypto_sign/empty/ref
            ```

        - ダミーコードの用意

            - crypto_sign/empty/ref/api.h

                ```c title="crypto_sign/empty/ref/api.h"
                #ifndef __API_H__
                #define __API_H__

                #define CRYPTO_SECRETKEYBYTES 64
                #define CRYPTO_PUBLICKEYBYTES 32
                #define CRYPTO_BYTES 64

                #endif
                ```

            - crypto_sign/empty/ref/sign.c

                ```c title="crypto_sign/empty/ref/sign.c"
                #include <string.h>
                #include "api.h"

                // 鍵ペア生成関数
                int crypto_sign_empty_ref_timingleaks_keypair(
                    unsigned char *pk,
                    unsigned char *sk)
                {
                    return 0;
                }

                // 署名生成関数
                int crypto_sign_empty_ref_timingleaks(
                    unsigned char *sm,
                    unsigned long long *smlen,
                    const unsigned char *m,
                    unsigned long long mlen,
                    const unsigned char *sk)
                {
                    return 0;
                }

                // 署名検証関数
                int crypto_sign_empty_ref_timingleaks_open(
                    unsigned char *m,
                    unsigned long long *mlen,
                    const unsigned char *sm,
                    unsigned long long smlen,
                    const unsigned char *pk)
                {
                    return 0;
                }
                ```

        - ダミーコードのビルドと実行

            ```bash
            ./do-part crypto_sign empty
            ```

            を実行すると下記のようなメッセージで終了すれば、コードのビルドと実行はできていて、crypto_sign_open 関数が予定の動作をしていない旨のメッセージが出力されます。

            ```text
            crypto_sign_open does not match mlen
            ```

## RSA2048

- ソース格納ディレクトリ作成

    ```bash
    mkdir -p crypto_sign/rsa2048/ref/
    ```

- crypto_sign/rsa2048/ref/api.h

    ```c title="crypto_sign/rsa2048/ref/api.h"
    #ifndef __API_H__
    #define __API_H__

    /* 秘密鍵・公開鍵・署名長のサイズ定義（DER を [長さ2B][本体] 格納） */

    #define CRYPTO_SECRETKEYBYTES  2048  /* 2 + DER(secret) を想定して十分大きく */
    #define CRYPTO_PUBLICKEYBYTES   512  /* 2 + DER(pub)    を想定して十分大きく */
    #define CRYPTO_BYTES            256  /* RSA 2048bit 署名長 */

    /* 関数プロトタイプ（NISTのAPIノート通り） */
    int crypto_sign_keypair(unsigned char *pk, unsigned char *sk);

    int crypto_sign(
        unsigned char *sm, unsigned long long *smlen,
        const unsigned char *m, unsigned long long mlen,
        const unsigned char *sk
    );

    int crypto_sign_open(
        unsigned char *m, unsigned long long *mlen,
        const unsigned char *sm, unsigned long long smlen,
        const unsigned char *pk
    );

    #endif
    ```

- crypto_sign/rsa2048/ref/sign.c

    ```c title="crypto_sign/rsa2048/ref/sign.c"
    #include <stdlib.h>
    #include <string.h>
    #include <stdint.h>

    #include <openssl/evp.h>
    #include <openssl/rsa.h>
    #include <openssl/pem.h>
    #include <openssl/err.h>

    #include "api.h"

    /*
    * timingleaks 用ラッパ関数
    */
    int crypto_sign_rsa2048_ref_timingleaks_keypair(
        unsigned char *pk,
        unsigned char *sk
    ) {
        return crypto_sign_keypair(pk, sk);
    }

    int crypto_sign_rsa2048_ref_timingleaks(
        unsigned char *sm, unsigned long long *smlen,
        const unsigned char *m, unsigned long long mlen,
        const unsigned char *sk
    ) {
        return crypto_sign(sm, smlen, m, mlen, sk);
    }

    int crypto_sign_rsa2048_ref_timingleaks_open(
        unsigned char *m, unsigned long long *mlen,
        const unsigned char *sm, unsigned long long smlen,
        const unsigned char *pk
    ) {
        return crypto_sign_open(m, mlen, sm, smlen, pk);
    }

    /* 2byte 長の読み書き用ヘルパ */
    static void store_u16_be(unsigned char *buf, uint16_t v)
    {
        buf[0] = (unsigned char)((v >> 8) & 0xff);
        buf[1] = (unsigned char)(v & 0xff);
    }

    static uint16_t load_u16_be(const unsigned char *buf)
    {
        return (uint16_t)((buf[0] << 8) | buf[1]);
    }

    /* 公開鍵：EVP_PKEY → pk バッファに [len][DER] 形式で保存 */
    static int write_pubkey(
        unsigned char *pk,
        EVP_PKEY *pkey
    ) {
        int len;
        unsigned char *der = NULL;
        unsigned char *p;

        len = i2d_PUBKEY(pkey, NULL);
        if (len <= 0 || len + 2 > CRYPTO_PUBLICKEYBYTES) {
            return -1;
        }

        p = der = (unsigned char *)OPENSSL_malloc(len);
        if (!der) return -1;

        if (i2d_PUBKEY(pkey, &p) != len) {
            OPENSSL_free(der);
            return -1;
        }

        store_u16_be(pk, (uint16_t)len);
        memcpy(pk + 2, der, len);

        OPENSSL_free(der);
        return 0;
    }
    
    /* pk バッファから EVP_PKEY を生成 */
    static EVP_PKEY *read_pubkey(const unsigned char *pk)
    {
        uint16_t len = load_u16_be(pk);
        const unsigned char *p = pk + 2;
        EVP_PKEY *pkey = NULL;

        if (2 + len > CRYPTO_PUBLICKEYBYTES) {
            return NULL;
        }

        pkey = d2i_PUBKEY(NULL, &p, len);
        return pkey; /* NULL の場合はエラー */
    }

    /* 秘密鍵：EVP_PKEY → sk バッファに [len][DER] 形式で保存 */
    static int write_privkey(unsigned char *sk, EVP_PKEY *pkey)
    {
        int len;
        unsigned char *der = NULL;
        unsigned char *p;

        /* 汎用秘密鍵 (PKCS#8) として保存 */
        len = i2d_PrivateKey(pkey, NULL);
        if (len <= 0 || len + 2 > CRYPTO_SECRETKEYBYTES) {
            return -1;
        }

        p = der = (unsigned char *)OPENSSL_malloc(len);
        if (!der) return -1;

        if (i2d_PrivateKey(pkey, &p) != len) {
            OPENSSL_free(der);
            return -1;
        }

        store_u16_be(sk, (uint16_t)len);
        memcpy(sk + 2, der, len);

        OPENSSL_free(der);
        return 0;
    }

    /* sk バッファから EVP_PKEY を生成 */
    static EVP_PKEY *read_privkey(const unsigned char *sk)
    {
        uint16_t len = load_u16_be(sk);
        const unsigned char *p = sk + 2;
        EVP_PKEY *pkey = NULL;

        if (2 + len > CRYPTO_SECRETKEYBYTES) {
            return NULL;
        }

        /* 中身を見て自動判定してくれる便利関数 */
        pkey = d2i_AutoPrivateKey(NULL, &p, len);
        return pkey; /* NULL の場合はエラー */
    }

    /*
     * crypto_sign_keypair
     *   - RSA 2048bit 鍵ペア生成
     *   - 公開鍵/秘密鍵をそれぞれ pk/sk バッファに格納
     */
    int crypto_sign_keypair(unsigned char *pk, unsigned char *sk)
    {
        int ret = -1;
        EVP_PKEY_CTX *ctx = NULL;
        EVP_PKEY *pkey = NULL;

        /* RSA キー生成コンテキスト作成 (RSA 2048bit) */
        ctx = EVP_PKEY_CTX_new_id(EVP_PKEY_RSA, NULL);
        if (!ctx) goto cleanup;

        if (EVP_PKEY_keygen_init(ctx) <= 0) goto cleanup;
        if (EVP_PKEY_CTX_set_rsa_keygen_bits(ctx, 2048) <= 0) goto cleanup;

        /* e=65537 はデフォルトで入るので pubexp の明示設定は省略してもよい */
        /*
        BIGNUM *e = BN_new();
        if (!e) goto cleanup;
        if (!BN_set_word(e, RSA_F4)) { BN_free(e); goto cleanup; }
        if (EVP_PKEY_CTX_set_rsa_keygen_pubexp(ctx, e) <= 0) {
            BN_free(e);
            goto cleanup;
        }
        BN_free(e);
        */

        if (EVP_PKEY_keygen(ctx, &pkey) <= 0) goto cleanup;

        /* バッファに保存 */
        if (write_pubkey(pk, pkey) != 0) goto cleanup;
        if (write_privkey(sk, pkey) != 0) goto cleanup;

        ret = 0;

    cleanup:
        if (pkey) EVP_PKEY_free(pkey);
        if (ctx) EVP_PKEY_CTX_free(ctx);

        return ret;
    }

    /*
     * crypto_sign
     *   入力:
     *     m, mlen : 元メッセージ
     *     sk      : 秘密鍵バイト列
     *   出力:
     *     sm      : [署名 256B] || [元メッセージ m]
     *     *smlen  : crypto_sign_BYTES + mlen
     */
    int crypto_sign(
        unsigned char *sm,
        unsigned long long *smlen,
        const unsigned char *m,
        unsigned long long mlen,
        const unsigned char *sk
    ) {
        int ret = -1;
        EVP_PKEY *pkey = NULL;
        EVP_MD_CTX *mdctx = NULL;
        size_t siglen = CRYPTO_BYTES;

        if (mlen + CRYPTO_BYTES > (unsigned long long)-1) {
            return -1; /* オーバーフロー防止 */
        }

        pkey = read_privkey(sk);
        if (!pkey) goto cleanup;

        mdctx = EVP_MD_CTX_new();
        if (!mdctx) goto cleanup;

        if (EVP_DigestSignInit(mdctx, NULL, EVP_sha256(), NULL, pkey) <= 0)
            goto cleanup;

        if (EVP_DigestSignUpdate(mdctx, m, (size_t)mlen) <= 0)
            goto cleanup;

        /* 署名長を取得・チェック */
        if (EVP_DigestSignFinal(mdctx, NULL, &siglen) <= 0)
            goto cleanup;
        if (siglen > CRYPTO_BYTES)
            goto cleanup;

        /* 実際に署名生成。sm 先頭に書き込む */
        if (EVP_DigestSignFinal(mdctx, sm, &siglen) <= 0)
            goto cleanup;

        /* 続けて元メッセージを連結 */
        memcpy(sm + CRYPTO_BYTES, m, (size_t)mlen);
        *smlen = (unsigned long long)CRYPTO_BYTES + mlen;

        ret = 0;

    cleanup:
        if (mdctx) EVP_MD_CTX_free(mdctx);
        if (pkey) EVP_PKEY_free(pkey);

        return ret;
    }

    /*
     * crypto_sign_open
     *   入力:
     *     sm, smlen : [署名 256B] || [元メッセージ]
     *     pk        : 公開鍵
     *   出力:
     *     m, *mlen  : 復元した元メッセージ
     *
     *   戻り値:
     *     0   : 検証成功
     *    -1等: 検証失敗
     */
    int crypto_sign_open(
        unsigned char *m,
        unsigned long long *mlen,
        const unsigned char *sm,
        unsigned long long smlen,
        const unsigned char *pk
    ) {
        int ret = -1;
        EVP_PKEY *pkey = NULL;
        EVP_MD_CTX *mdctx = NULL;
        const unsigned char *sig;
        const unsigned char *msg;
        unsigned long long msglen;

        if (smlen < CRYPTO_BYTES) {
            return -1; /* 不正な長さ */
        }

        sig = sm;                      /* 先頭 crypto_sign_BYTES が署名 */
        msg = sm + CRYPTO_BYTES;       /* その後ろがメッセージ本体 */
        msglen = smlen - CRYPTO_BYTES;

        pkey = read_pubkey(pk);
        if (!pkey) goto cleanup;

        mdctx = EVP_MD_CTX_new();
        if (!mdctx) goto cleanup;

        if (EVP_DigestVerifyInit(mdctx, NULL, EVP_sha256(), NULL, pkey) <= 0)
            goto cleanup;

        if (EVP_DigestVerifyUpdate(mdctx, msg, (size_t)msglen) <= 0)
            goto cleanup;

        if (EVP_DigestVerifyFinal(mdctx, sig, CRYPTO_BYTES) != 1) {
            /* 署名検証 NG */
            ret = -1;
            goto cleanup;
        }

        /* 検証 OK の場合、メッセージを呼び出し側に返す */
        memcpy(m, msg, (size_t)msglen);
        *mlen = msglen;
        ret = 0;

    cleanup:
        if (mdctx) EVP_MD_CTX_free(mdctx);
        if (pkey) EVP_PKEY_free(pkey);

        return ret;
    }
    ```

- 測定の実行

    環境変数を設定しないと libcrypto.so (OpenSSL のライブラリ) が /lib/x86_64-linux-gnu/libcrypto.so.3 を参照してしまい、ソースからビルドした方 (/usr/local/lib64/libcrypto.so.3) を参照してくれないので注意が必要です。

    ```bash
    # 独自ビルドした libcrypto.so.3 を強制的に最優先でロード
    export LD_PRELOAD=/usr/local/lib64/libcrypto.so.3 
    # その他環境変数設定
    export OPENSSL_CONF=/dev/null
    export OPENSSL_MODULES=/usr/local/lib64/ossl-modules/
    export LD_LIBRARY_PATH=/usr/local/lib:/usr/local/lib64:$LD_LIBRARY_PATH

    ./do-part crypto_sign rsa2048
    ```

- リンクしているライブラリの確認

    ```bash
    ldd ./bench/*/work/compile/measure
    ldd ./bench/*/work/compile/try
    ldd ./bench/*/work/compile/try-small
    ```

## ML_DSA_44 (まだ動かない)

- ソース格納ディレクトリ作成

    ```bash
    mkdir -p crypto_sign/mldsa44/ref/
    ```

- crypto_sign/mldsa44/ref/api.h

    ```c title="crypto_sign/mldsa44/ref/api.h"
    #ifndef __API_H__
    #define __API_H__

    // ML-DSA-44 のサイズ（OQS 定義と一致）
    #define CRYPTO_PUBLICKEYBYTES 1312
    #define CRYPTO_SECRETKEYBYTES 2528
    #define CRYPTO_BYTES 2420

    #endif
    ```

- crypto_sign/mldsa44/ref/sign.c

    ```c title="crypto_sign/mldsa44/ref/sign.c"
    #include <string.h>
    #include "api.h"

    // フルパス include
    #include "openssl/evp.h"
    #include "openssl/provider.h"

    // 静的ライブラリのリンク指示（GCC 向け）
    __asm__(".pushsection .linker_opts\n.string \"-L./lib -lcrypto -loqs\"\n.popsection");

    // ---------------------------------------------
    // キーペア生成
    // ---------------------------------------------
    int crypto_sign_mldsa44_ref_timingleaks_keypair(
        unsigned char *pk,
        unsigned char *sk)
    {
        int ret = -1;
        EVP_PKEY_CTX *pctx = NULL;
        EVP_PKEY *pkey = NULL;

        OSSL_PROVIDER_load(NULL, "oqsprovider");

        pctx = EVP_PKEY_CTX_new_from_name(NULL, "ML-DSA-44", NULL);
        if (!pctx) goto end;
        if (EVP_PKEY_keygen_init(pctx) <= 0) goto end;
        if (EVP_PKEY_generate(pctx, &pkey) <= 0) goto end;

        size_t sk_len = CRYPTO_SECRETKEYBYTES;
        size_t pk_len = CRYPTO_PUBLICKEYBYTES;

        if (EVP_PKEY_get_raw_private_key(pkey, sk, &sk_len) <= 0) goto end;
        if (EVP_PKEY_get_raw_public_key(pkey, pk, &pk_len) <= 0) goto end;

        ret = 0;

    end:
        EVP_PKEY_free(pkey);
        EVP_PKEY_CTX_free(pctx);
        return ret;
    }

    // ---------------------------------------------
    // 署名生成
    // ---------------------------------------------
    int crypto_sign_mldsa44_ref_timingleaks(
        unsigned char *sm,
        unsigned long long *smlen,
        const unsigned char *m,
        unsigned long long mlen,
        const unsigned char *sk)
    {
        int ret = -1;
        EVP_PKEY *pkey = NULL;
        EVP_MD_CTX *mdctx = NULL;

        OSSL_PROVIDER_load(NULL, "oqsprovider");

        pkey = EVP_PKEY_new_raw_private_key(EVP_PKEY_NONE, NULL, sk, CRYPTO_SECRETKEYBYTES);
        if (!pkey) goto end;

        mdctx = EVP_MD_CTX_new();
        if (!mdctx) goto end;

        if (EVP_DigestSignInit(mdctx, NULL, NULL, NULL, pkey) <= 0) goto end;
        if (EVP_DigestSignUpdate(mdctx, m, mlen) <= 0) goto end;

        size_t siglen = CRYPTO_BYTES;
        if (EVP_DigestSignFinal(mdctx, sm, &siglen) <= 0) goto end;

        *smlen = siglen;
        ret = 0;

    end:
        EVP_MD_CTX_free(mdctx);
        EVP_PKEY_free(pkey);
        return ret;
    }

    // ---------------------------------------------
    // 署名検証
    // ---------------------------------------------
    int crypto_sign_mldsa44_ref_timingleaks_open(
        unsigned char *m,
        unsigned long long *mlen,
        const unsigned char *sm,
        unsigned long long smlen,
        const unsigned char *pk)
    {
        int ret = -1;
        EVP_PKEY *pkey = NULL;
        EVP_MD_CTX *mdctx = NULL;

        OSSL_PROVIDER_load(NULL, "oqsprovider");

        pkey = EVP_PKEY_new_raw_public_key(EVP_PKEY_NONE, NULL, pk, CRYPTO_PUBLICKEYBYTES);
        if (!pkey) goto end;

        mdctx = EVP_MD_CTX_new();
        if (!mdctx) goto end;

        if (EVP_DigestVerifyInit(mdctx, NULL, NULL, NULL, pkey) <= 0) goto end;
        if (EVP_DigestVerifyUpdate(mdctx, m, *mlen) <= 0) goto end;

        if (EVP_DigestVerifyFinal(mdctx, sm, smlen) != 1) goto end;

        ret = 0;

    end:
        EVP_MD_CTX_free(mdctx);
        EVP_PKEY_free(pkey);
        return ret;
    }
    ```

- 測定の実行

    ```bash
    ./do-part crypto_sign mldsa44
    ```
