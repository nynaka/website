# Web3Signer 署名鍵の HashiCorp Vault 保存

Web3Signer の署名鍵を HashiCorp Vault を使って保存する手順を説明します。

## 動作環境

|                 |                 |
| :-------------- | :-------------- |
| OS              | Ubuntu 24.04    |
| JDK             | Open JDK 21.0.9 |
| Web3Signer      | 25.12.0         |
| Hashicorp Vault | 1.16.2          |

## HashiCorp Vault のセットアップ

### Vault のインストール

HashiCorp の公式リポジトリを登録して Vault をインストールします。

```bash
# GPG キーの追加
wget -O- https://apt.releases.hashicorp.com/gpg \
    | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
# リポジトリの追加
echo \
    "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
    | sudo tee /etc/apt/sources.list.d/hashicorp.list
# パッケージのインストール
sudo apt update
sudo apt install -y vault
```

### Vault 開発サーバーの起動

テスト用に `dev` モードで Vault サーバーを起動します。

```bash
vault server -dev
```

このコマンドを実行すると、アンシールキーと **Root Token** が表示されます。このトークンは後で使いますので、メモしておいてください。

```text
... 
Root Token: hvs.***************************
... 
```

**別のターミナルを開いて**、以下の作業を続けます。Vault サーバーは起動したままにしておきます。

### Vault 環境変数の設定

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
# サーバー起動時に表示された Root Token を設定
export VAULT_TOKEN='hvs.***************************'
```

## 署名鍵の作成と Vault への保存

### 秘密鍵の生成

`openssl` を使って `secp256k1` の秘密鍵を生成します。

```bash
openssl ecparam -name secp256k1 -genkey -noout -out my-eth-key.pem
# 16進数形式に変換 (先頭の "0x" は付けない)
PRIVATE_KEY=$(openssl ec -in my-eth-key.pem -text -noout | grep "priv:" -A 3 | tail -n +2 | tr -d '[:space:]:' | sed 's/^00//')
echo $PRIVATE_KEY
```

このコマンドで出力された長い16進数の文字列が秘密鍵です。

### Vault への秘密鍵の保存

`kv` secrets engine を使って、生成した秘密鍵を Vault に保存します。

```bash
vault kv put secret/web3signer/my-eth-key value=$PRIVATE_KEY
```

## Web3Signer の設定

### Web3Signer のインストール (未実施の場合)

- 必要なパッケージのインストール

    ```bash
    sudo apt install -y openjdk-21-jdk
    ```

- インストール

    ```bash
    wget https://github.com/Consensys/web3signer/releases/download/25.12.0/web3signer-25.12.0.tar.gz
    tar zxvf web3signer-25.12.0.tar.gz -C $HOME
    echo 'export PATH=$HOME/web3signer-25.12.0/bin:$PATH' >> $HOME/.bashrc
    source $HOME/.bashrc
    ```

### 署名鍵の設定ファイル作成

Web3Signer に HashiCorp Vault を使うよう設定します。

```bash
mkdir -p $HOME/web3signer/keys
```

設定ファイル (`vault-key.yaml`) を作成します。

```yaml title="$HOME/web3signer/keys/vault-key.yaml"
type: "hashicorp"
keyType: "SECP256K1"
serverHost: "127.0.0.1"
serverPort: 8200
timeout: 10000
tlsEnabled: false
token: "hvs.***************************" # Vault の Root Token
keyPath: "/secret/data/web3signer/my-eth-key"
keyName: "value"
```

- `token` はご自身の Vault トークンに置き換えてください。
- `keyPath` は `kv` secrets engine のバージョン2 (デフォルト) を使う場合、`secret/data/...` というパスになります。

## Web3Signer の起動と署名

### Web3Signer の起動

```bash
web3signer --key-store-path=$HOME/web3signer/keys eth1 --chain-id=31337
```

起動すると、Web3Signer が Vault から秘密鍵を読み込み、対応する Ethereum アドレスをログに出力します。

```log
... 
INFO  - Loading keys from Hashicorp Vault configuration: serverHost=127.0.0.1, serverPort=8200, keyPath=/secret/data/web3signer/my-eth-key, keyName=value
... 
INFO  - Address: 0x...
... 
```
この `Address` が署名に使う Ethereum アドレスです。

### 署名の実行

前のドキュメントと同様に `curl` を使って署名リクエストを送ります。

```json
curl -X POST http://localhost:9000 \
    -H "Content-Type: application/json" \
    -d \
    '{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sign",
  "params": ["<YOUR-ETHEREUM-ADDRESS>", "0x7d3ec8fba8a1b42b74e7c0cbab2a73dbf9a84e13f96e3e8f4fd09940420e7a6f"]
}'
```

`<YOUR-ETHEREUM-ADDRESS>` の部分を、Web3Signer の起動ログに表示されたアドレスに置き換えてください。

成功すると、署名データを含んだJSONレスポンスが返ってきます。

```json
{
  "jsonrpc" : "2.0",
  "id" : 1,
  "result" : "0x..."
}
```
これで、HashiCorp Vault に保存された秘密鍵を使って署名ができました。
