Web3signer on Hardhat
===

## 必要なパッケージのインストール

```bash
sudo apt install -y \
    openjdk-21-jdk \
    nodejs npm
```

## Hardhat

### インストール

```bash
npm install hardhat
```

### 初期化と起動

- 初期化

    ```bash
    npx hardhat init
    ```

- 起動

    ```bash
    npx hardhat node
    ```

    Hardhat は起動ごとに 10,000 ETH の資産を持ったアカウントを 20 個作成してくれます。


## Web 3 Signer

```bash
git clone https://github.com/Consensys/web3signer.git
cd web3signer
```

```bash
./gradle
```




