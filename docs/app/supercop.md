SUPERCOP
===

## インストール

### 依存パッケージ

```bash
sudo apt update
sudo apt install -y build-essential git curl python3 \
    libssl-dev pkg-config make autoconf automake
```

### SUPERCOP の入手と初期化

- [開発元サイト](https://bench.cr.yp.to/supercop.html) の手順

    ```bash
    wget https://bench.cr.yp.to/supercop/supercop-20251222.tar.xz
    tar Jxf supercop-20251222.tar.xz
    cd supercop-20251222

    # 環境の初期化 (概ね30分くらい待つ。省略しても問題ない。)
    time nohup ./do
    ```

- Github

    ```bash
    # 置き場所は任意
    mkdir -p ~/bench \
        && cd ~/bench
    git clone https://github.com/jedisct1/supercop.git
    cd supercop

    # 環境の初期化 (概ね30分くらい待つ。省略しても問題ない。)
    time nohup ./do
    ```

### 性能測定の安定化

!!! note
    必須の手順ではないが、測定結果が安定するらしい。  
    クラウドサービスや仮想マシンの場合、エラーになることがある。

```bash
# Intel のターボブースト OFF
echo 1 | sudo tee /sys/devices/system/cpu/intel_pstate/no_turbo

# AMD の boost OFF
echo 0 | sudo tee /sys/devices/system/cpu/cpufreq/boost

# 全コアを "performance" governor に
for i in /sys/devices/system/cpu/cpu*/cpufreq; do
  echo performance | sudo tee "$i/scaling_governor"
done
```

### コンパイルパラメータの調整

**okcompilers/c** で -O3 以外の行をコメントにする。

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

### 測定環境の初期化

```bash
./do-part init
./do-part crypto_verify 32
./do-part crypto_hash sha512
./do-part crypto_stream chacha20
./do-part crypto_rng chacha20
./do-part crypto_sign ed25519
```

エラーが出ると思いますが、あまり重要ではないので無視します。  
ここで重要なのは **./bench/ホスト名ディレクトリ/** が作成され、その中にいろいろなファイルが生成されることになります。


## 測定の実行

!!! note
    ここでは RSA3072 のキーペア作成、署名、署名検証の処理性能測定を例にします。  
    測定用コード格納ディレクトリは **crypto_sign/rsa3072/ref/** とします。

### 測定用コードの用意

```c title="crypto_sign/rsa3072/ref/api.h"
#ifndef __API_H__
#define __API_H__

#include <stddef.h>
#include <stdint.h>

/* Sizes (conservative fixed buffers) */
#define CRYPTO_PUBLICKEYBYTES 1200
#define CRYPTO_SECRETKEYBYTES 2400
#define CRYPTO_BYTES 384

/* SUPERCOP-style API (attached-signature form)
   - crypto_sign_keypair(pk, sk)
   - crypto_sign(sm, smlen, m, mlen, sk)
   - crypto_sign_open(m, mlen, sm, smlen, pk)
 */

int crypto_sign_keypair(unsigned char *pk, unsigned char *sk);
int crypto_sign(unsigned char *sm, unsigned long long *smlen,
                const unsigned char *m, unsigned long long mlen,
                const unsigned char *sk);
int crypto_sign_open(unsigned char *m, unsigned long long *mlen,
                     const unsigned char *sm, unsigned long long smlen,
                     const unsigned char *pk);

/* SUPERCOP symbol aliases: map expected implementation names to generic APIs */
/* These macros let the SUPERCOP build use the generic function names at compile time
  so we don't need separate wrapper functions for each security model. */

/* timingleaks variant */
#define crypto_sign_rsa3072_ref_timingleaks_keypair crypto_sign_keypair
#define crypto_sign_rsa3072_ref_timingleaks crypto_sign
#define crypto_sign_rsa3072_ref_timingleaks_open crypto_sign_open

#endif /* __API_H__ */
```

```c title="crypto_sign/rsa3072/ref/sign.c"
#include <openssl/err.h>
#include <openssl/evp.h>
#include <openssl/pem.h>
#include <openssl/rsa.h>
#include <openssl/sha.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

#include "api.h"

/* Helper: store length (uint32_t) prefix in buffers */
static void store_len(unsigned char* buf, uint32_t len)
{
	buf[0] = (len >> 24) & 0xff;
	buf[1] = (len >> 16) & 0xff;
	buf[2] = (len >> 8) & 0xff;
	buf[3] = (len) & 0xff;
}
static uint32_t load_len(const unsigned char* buf)
{
	return ((uint32_t)buf[0] << 24) | ((uint32_t)buf[1] << 16) |
		   ((uint32_t)buf[2] << 8) | (uint32_t)buf[3];
}


int crypto_sign_keypair(unsigned char* pk, unsigned char* sk)
{
	int ret = -1;
	RSA* rsa = NULL;
	BIGNUM* e = NULL;
	unsigned char* pder = NULL;
	unsigned char* sder = NULL;
	int plen = 0, slen = 0;

	e = BN_new();
	if (!e) goto done;
	if (!BN_set_word(e, RSA_F4)) goto done;
	rsa = RSA_new();
	if (!rsa) goto done;
	if (!RSA_generate_key_ex(rsa, 3072, e, NULL)) goto done;

	/* public DER */
	plen = i2d_RSA_PUBKEY(rsa, NULL);
	if (plen <= 0 || plen + 4 > CRYPTO_PUBLICKEYBYTES) goto done;
	pder = malloc(plen);
	if (!pder) goto done;
	unsigned char* pderp = pder;
	if (i2d_RSA_PUBKEY(rsa, &pderp) != plen) goto done;

	/* private DER */
	slen = i2d_RSAPrivateKey(rsa, NULL);
	if (slen <= 0 || slen + 4 > CRYPTO_SECRETKEYBYTES) goto done;
	sder = malloc(slen);
	if (!sder) goto done;
	unsigned char* sderp = sder;
	if (i2d_RSAPrivateKey(rsa, &sderp) != slen) goto done;

	/* Store length-prefixed blobs */
	store_len(pk, (uint32_t)plen);
	memcpy(pk + 4, pder, plen);
	memset(pk + 4 + plen, 0, CRYPTO_PUBLICKEYBYTES - 4 - plen);

	store_len(sk, (uint32_t)slen);
	memcpy(sk + 4, sder, slen);
	memset(sk + 4 + slen, 0, CRYPTO_SECRETKEYBYTES - 4 - slen);

	ret = 0;

done:
	if (pder) free(pder);
	if (sder) free(sder);
	if (rsa) RSA_free(rsa);
	if (e) BN_free(e);
	return ret;
}

int crypto_sign(unsigned char* sm, unsigned long long* smlen,
				const unsigned char* m, unsigned long long mlen,
				const unsigned char* sk)
{
	unsigned char digest[SHA256_DIGEST_LENGTH];
	unsigned int siglen = 0;
	int ret = -1;
	uint32_t slen = load_len(sk);
	const unsigned char* sder = sk + 4;
	const unsigned char* p = sder; /* d2i modifies pointer */
	RSA* rsa = d2i_RSAPrivateKey(NULL, (unsigned char**)&p, slen);
	if (!rsa) return -1;

	/* compute hash */
	if (!SHA256(m, (size_t)mlen, digest)) goto done;

	unsigned char* sig = malloc(RSA_size(rsa));
	if (!sig) goto done;
	if (!RSA_sign(NID_sha256, digest, SHA256_DIGEST_LENGTH, sig, &siglen,
				  rsa)) {
		free(sig);
		goto done;
	}

	/* attach signature || message */
	if (siglen > CRYPTO_BYTES) {
		free(sig);
		goto done;
	}
	memcpy(sm, sig, siglen);
	memcpy(sm + siglen, m, (size_t)mlen);
	*smlen = (unsigned long long)(siglen + mlen);
	free(sig);
	ret = 0;

done:
	RSA_free(rsa);
	return ret;
}

int crypto_sign_open(unsigned char* m, unsigned long long* mlen,
					 const unsigned char* sm, unsigned long long smlen,
					 const unsigned char* pk)
{
	if (smlen < 1) return -1;
	uint32_t plen = load_len(pk);
	const unsigned char* pder = pk + 4;
	const unsigned char* p = pder;
	RSA* rsa = d2i_RSA_PUBKEY(NULL, (unsigned char**)&p, plen);
	if (!rsa) return -1;

	/* signature length is RSA_size (3072/8 = 384) — we use CRYPTO_BYTES */
	if (smlen < CRYPTO_BYTES) {
		RSA_free(rsa);
		return -1;
	}
	unsigned int siglen = CRYPTO_BYTES;
	const unsigned char* sig = sm;
	const unsigned char* msg = sm + siglen;
	unsigned long long msglen = smlen - siglen;

	unsigned char digest[SHA256_DIGEST_LENGTH];
	if (!SHA256(msg, (size_t)msglen, digest)) {
		RSA_free(rsa);
		return -1;
	}

	int ok =
		RSA_verify(NID_sha256, digest, SHA256_DIGEST_LENGTH, sig, siglen, rsa);
	if (!ok) {
		RSA_free(rsa);
		*mlen = (unsigned long long)-1;
		return -1;
	}

	memcpy(m, msg, (size_t)msglen);
	*mlen = msglen;
	RSA_free(rsa);
	return 0;
}
```

### 測定の実行

```bash
./do-part crypto_sign rsa3072
```

### 測定結果の集計

測定結果は CPU のサイクルで出力されるため、秒に変換します。  
gawk がインストールされていないとエラーになります。

- 変換スクリプト

    ```bash title="summary.sh"
    #!/usr/bin/env bash

    LOGFILE="bench/ubuntu2404a/data"

    # CPU cycles/sec を取得
    CPS=$(grep 'cpucycles_persecond' "$LOGFILE" | awk '{print $NF}' | head -n 1)

    if [ -z "$CPS" ]; then
        echo "Error: cpucycles_persecond がログから取得できませんでした。" >&2
        exit 1
    fi

    echo "Detected CPU cycles/sec = $CPS"
    echo

    # 指定パターンにマッチする全行の 9〜24列 cycles を集め、
    # 全体の中央値を求める関数
    compute_median() {
    local pattern="$1"

    grep "$pattern" "$LOGFILE" | gawk -v cps="$CPS" '
        {
        # 各行の 16 サンプルを a[] に格納
        for (i = 9; i <= 24; i++) {
            samples[n++] = $i
        }
        }
        END {
        if (n == 0) {
            exit 1
        }
        # 全部まとめてソート
        asort(samples)

        # 0-based → 中央値は (n-1)/2
        median = samples[int((n-1)/2)]
        sec = median / cps

        printf "%.0f %.6f\n", median, sec
        }
    '
    }

    # 実際の出力処理
    print_stat() {
    local label="$1"
    local desc="$2"
    local pattern="$3"

    local out
    out=$(compute_median "$pattern") || {
        printf "%-10s %-25s Not found (pattern: %s)\n" "[$label]" "$desc" "$pattern"
        return
    }

    local median sec
    median=$(echo "$out" | awk '{print $1}')
    sec=$(echo    "$out" | awk '{print $2}')

    printf "%-10s %-25s median cycles = %-12s median time = %s sec\n" \
        "[$label]" "$desc" "$median" "$sec"
    }

    echo "==== crypto_sign rsa3072/timingleaks summary (ALL SAMPLES) ===="

    # 鍵生成（全 keypair_cycles 行）
    print_stat "keypair" "key generation" \
    "crypto_sign rsa3072/timingleaks keypair_cycles"

    # 署名（全 cycles <index> 行 = `... timingleaks cycles <数字>`）
    print_stat "sign" "sign (all cycles)" \
    "crypto_sign rsa3072/timingleaks cycles [0-9]"

    # 署名検証（全 open_cycles <index> 行）
    print_stat "verify" "verify (all open_cycles)" \
    "crypto_sign rsa3072/timingleaks open_cycles [0-9]"

    echo "==============================================================="
    ```

- 実行例

    ```bash
    # bash ./summary.sh
    Detected CPU cycles/sec = 2995199000

    ==== crypto_sign rsa3072/timingleaks summary (ALL SAMPLES) ====
    [keypair]  key generation            median cycles = 2050700603   median time = 0.684663 sec
    [sign]     sign (all cycles)         median cycles = 5488297      median time = 0.001832 sec
    [verify]   verify (all open_cycles)  median cycles = 205130       median time = 0.000068 sec
    ===============================================================
    ```

    !!! note
		データの集計や現状把握には秒数を用いるのが直感的ですが、公開されているベンチマーク等の外部データと今回の測定結果を比較し、妥当性を検証する際にはCPUサイクル数が指標となるケースが多いため、両方の値を記録しておくことが有用です。
