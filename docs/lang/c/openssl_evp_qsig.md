OpenSSL EVPインタフェースを使用した耐量子暗号アルゴリズム署名
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

-  作業用ディレクトリの作成

    作業ディレクトリを $HOME/work とし、その下の chroot にライブラリやヘッダファイルを格納することを想定しています。

    ```bash
    cd ~
    mkdir -p work/chroot
    cd work
    export INSTALL_DIR=$(pwd)/chroot
    ```

- OpenSSL のインストール

    ```bash
    wget https://github.com/openssl/openssl/archive/refs/tags/openssl-3.6.0.tar.gz
    tar zxf openssl-3.6.0.tar.gz
    cd openssl-openssl-3.6.0/
    ./config --prefix=$INSTALL_DIR --openssldir=$INSTALL_DIR/ssl
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
    cmake -GNinja \
        -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR \
        -DBUILD_SHARED_LIBS=ON \
        -DOQS_USE_OPENSSL=OFF \
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

## サンプルコードのビルドと実行

- サンプルコード

    ```c title="mldsa44_sample.c"
    #include <stdio.h>
    #include <string.h>
    #include <openssl/provider.h>
    #include <openssl/evp.h>
    #include <openssl/err.h>

    // エラーハンドリング用マクロ
    #define HANDLE_ERROR() \
        do { \
            fprintf(stderr, "Error at line %d\n", __LINE__); \
            ERR_print_errors_fp(stderr); \
            return 1; \
        } while (0)

    int main() {
        OSSL_PROVIDER *oqsprov = NULL;
        OSSL_PROVIDER *defprov = NULL;
        EVP_PKEY_CTX *pctx = NULL;
        EVP_PKEY *pkey = NULL;
        EVP_MD_CTX *mdctx = NULL;

        const char *alg_name = "mldsa44"; // ML-DSA-44 アルゴリズム名
        const char *message = "This is a message signed by ML-DSA-44.";
        unsigned char *sig = NULL;
        size_t sig_len = 0;
        int ret = 1; // 終了コード (0: 成功, 1: 失敗)

        printf("--- OpenSSL EVP with ML-DSA-44 Sample ---\n");

        // 1. プロバイダーのロード
        // oqsprovider (PQCアルゴリズム用)
        oqsprov = OSSL_PROVIDER_load(NULL, "oqsprovider");
        if (oqsprov == NULL) {
            fprintf(stderr, "Failed to load oqsprovider\n");
            HANDLE_ERROR();
        }
        // defaultプロバイダー (乱数生成等の基本機能用)
        defprov = OSSL_PROVIDER_load(NULL, "default");
        if (defprov == NULL) {
            fprintf(stderr, "Failed to load default provider\n");
            HANDLE_ERROR();
        }
        printf("[Success] Providers loaded.\n");

        // 2. 鍵ペアの生成 (ML-DSA-44)
        // コンテキストの作成
        pctx = EVP_PKEY_CTX_new_from_name(NULL, alg_name, NULL);
        if (pctx == NULL) HANDLE_ERROR();

        // 鍵生成の初期化
        if (EVP_PKEY_keygen_init(pctx) <= 0) HANDLE_ERROR();

        // 鍵生成の実行
        printf("Generating Key Pair (%s)...\n", alg_name);
        if (EVP_PKEY_keygen(pctx, &pkey) <= 0) HANDLE_ERROR();
        printf("[Success] Key Pair generated.\n");

        // 3. 署名の作成
        mdctx = EVP_MD_CTX_new();
        if (mdctx == NULL) HANDLE_ERROR();

        // 署名コンテキストの初期化 (ML-DSAはダイジェストを指定せずNULLにするのが一般的)
        if (EVP_DigestSignInit(mdctx, NULL, NULL, NULL, pkey) <= 0) HANDLE_ERROR();

        // 署名サイズの取得 (1回目)
        if (EVP_DigestSign(mdctx, NULL, &sig_len, (const unsigned char *)message, strlen(message)) <= 0) HANDLE_ERROR();

        // メモリ確保
        sig = OPENSSL_malloc(sig_len);
        if (sig == NULL) HANDLE_ERROR();

        // 署名の実行 (2回目)
        if (EVP_DigestSign(mdctx, sig, &sig_len, (const unsigned char *)message, strlen(message)) <= 0) HANDLE_ERROR();

        printf("[Success] Message signed. Signature size: %zu bytes.\n", sig_len);

        // 4. 署名の検証
        // 検証用にコンテキストをリセット（または再作成）
        EVP_MD_CTX_reset(mdctx);

        if (EVP_DigestVerifyInit(mdctx, NULL, NULL, NULL, pkey) <= 0) HANDLE_ERROR();

        printf("Verifying signature...\n");
        int verify_result = EVP_DigestVerify(mdctx, sig, sig_len, (const unsigned char *)message, strlen(message));

        if (verify_result == 1) {
            printf("\n[RESULT] Signature Verification: OK (Success)\n");
            ret = 0;
        } else {
            printf("\n[RESULT] Signature Verification: FAILED\n");
            HANDLE_ERROR();
        }

        // クリーンアップ
        OPENSSL_free(sig);
        EVP_MD_CTX_free(mdctx);
        EVP_PKEY_free(pkey);
        EVP_PKEY_CTX_free(pctx);
        OSSL_PROVIDER_unload(oqsprov);
        OSSL_PROVIDER_unload(defprov);

        return ret;
    }
    ```

- ビルド

    ```bash
    # パスの設定
    export INSTALL_DIR=$HOME/work/chroot
    export LD_LIBRARY_PATH=$HOME/work/chroot/lib:$HOME/work/chroot/lib64:$LD_LIBRARY_PATH

    # ビルド
    gcc -o mldsa44_sample mldsa44_sample.c \
        -I$INSTALL_DIR/include \
        -L$INSTALL_DIR/lib64 \
        -lssl -lcrypto -Wl,-rpath,$INSTALL_DIR/lib64
    ```

- 実行結果

    ```bash
    # ./mldsa44_sample
    --- OpenSSL EVP with ML-DSA-44 Sample ---
    [Success] Providers loaded.
    Generating Key Pair (mldsa44)...
    [Success] Key Pair generated.
    [Success] Message signed. Signature size: 2420 bytes.
    Verifying signature...

    [RESULT] Signature Verification: OK (Success)
    ```
