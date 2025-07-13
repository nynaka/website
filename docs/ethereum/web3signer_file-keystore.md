Web3Signer の署名動作確認
===

## 動作環境

|            |                                     |
| :--------- | :---------------------------------- |
| OS         | Ubuntu 24.04 (WSL Docker Container) |
| JDK        | Open JDK 21.0.7                     |
| Web3Signer | 25.6.0                              |


## keystore 形式のアカウント作成

- go のインストール

    ```bash
    sudo apt install -y golang
    ```

- geth コマンドのインストール

    - ビルド

        ```bash
        git clone https://github.com/ethereum/go-ethereum.git
        cd go-ethereum/
        make geth
        ```

    - インストール

        ```bash
        mkdir $HOME/bin
        cp ./build/bin/geth $HOME/bin
        ```

    - PATH の設定

        ```bash
        echo 'PATH=$HOME/bin:$PATH' >> $HOME/.bashrc
        source $HOME/.bashrc
        ```

- keystore 形式のアカウント作成

    ```bash
    mkdir -p $HOME/web3signer/keystores
    geth account new --keystore $HOME/web3signer/keystores
    ```

    パスワードを設定すると下記のように作成したアカウントの情報が表示されます。

    ```text
    Your new key was generated

    Public address of the key:   0xF7ac504BAc5De36C03483586FCdd11cA97f604F8
    Path of the secret key file: /home/youruser/web3signer/keystores/UTC--2025-07-12T12-52-35.515154612Z--f7ac504bac5de36c03483586fcdd11ca97f604f8

    - You can share your public address with anyone. Others need it to interact with you.
    - You must NEVER share the secret key with anyone! The key controls access to your funds!
    - You must BACKUP your key file! Without the key, it's impossible to access account funds!
    - You must REMEMBER your password! Without the password, it's impossible to decrypt the key!
    ```


## Web3Signer のインストール

### 必要なパッケージのインストール

```bash
sudo apt install -y openjdk-21-jdk
```

### インストール

- [Install binary distribution](https://docs.web3signer.consensys.io/get-started/install-binaries)

    - ビルド済みバイナリの取得と展開

        ```bash
        wget https://github.com/Consensys/web3signer/releases/download/25.6.0/web3signer-25.6.0.tar.gz
        tar zxvf web3signer-25.6.0.tar.gz -C $HOME
        ```

    - PATH の設定

        ```bash
        echo 'export PATH=$HOME/web3signer-25.6.0/bin:$PATH' >> $HOME/.bashrc
        source $HOME/.bashrc
        ```


### 署名鍵の設定

- 署名鍵設定ファイルディレクトリ作成

    ```bash
    mkdir $HOME/web3signer/keys
    ```

- パスワードファイルの作成

    ```bash
    echo "password" > $HOME/web3signer/f7ac504bac5de36c03483586fcdd11ca97f604f8.password
    ```

- 署名鍵の設定

    ```yaml title="$HOME/web3signer/keys/localkey.yaml"
    type: "file-keystore"
    keyType: "SECP256K1"
    keystoreFile: "/home/youruser/web3signer/keystores/UTC--2025-07-12T12-52-35.515154612Z--f7ac504bac5de36c03483586fcdd11ca97f604f8"
    keystorePasswordFile: "/home/youruser/web3signer/f7ac504bac5de36c03483586fcdd11ca97f604f8.password"
    ```

- Web3Signer の起動

    ```bash
    web3signer --key-store-path=$HOME/web3signer/keys eth1 --chain-id=31337
    ```

### 署名の実行

```json
curl -X POST http://localhost:9000 \
    -H "Content-Type: application/json" \
    -d '
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "eth_sign",
  "params": ["0xf7ac504bac5de36c03483586fcdd11ca97f604f8", "0x7d3ec8fba8a1b42b74e7c0cbab2a73dbf9a84e13f96e3e8f4fd09940420e7a6f"]
}'
```

params で設定している配列要素の 1 個目のパラメータに **Public address of the key** の値を全部小文字にして先頭に **0x** を付けた値を設定します。


## その他の Web3Signer のAPI

[Consensys Web3Signer API documentation](https://consensys.github.io/web3signer/) に Web3Signer が提供する API の説明が書かれています。  
上記の署名 API は古い形式で、新しい署名 API では API documentation に書かれている形式を指定するらしいのですが、どうしても **400 Bad request format** の壁を越えられなかった。。。あとで時間があるときにリトライしようと思います。