Ubuntu 24.04 (WSL)
===

## CLI ツール

```bash
sudo apt update
sudo apt install -y \
    ssh \
    vim git man manpages-ja \
    curl jq wget zip unzip bzip2 p7zip-full \
    python3-venv
```

## [Docker](https://docs.docker.com/engine/install/ubuntu/)

* 関連ツールのインストール

    ```bash
    sudo apt-get update
    sudo apt-get -y install \
        ca-certificates \
        curl
    ```

* GPG key のインストール

    ```bash
    sudo install -m 0755 -d /etc/apt/keyrings
    sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
        -o /etc/apt/keyrings/docker.asc
    sudo chmod a+r /etc/apt/keyrings/docker.asc
    ```

* リポジトリの追加

    ```bash
    sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
    Types: deb
    URIs: https://download.docker.com/linux/ubuntu
    Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
    Components: stable
    Signed-By: /etc/apt/keyrings/docker.asc
    EOF
    ```

* Docker のインストール
    ```bash
    sudo apt-get update
    sudo apt install -y \
        docker-ce docker-ce-cli \
        containerd.io docker-buildx-plugin \
        docker-compose-plugin
    ```

* ユーザ権限の設定

    ```bash
    sudo usermod -aG docker ubuntu
    ```

* 自動起動設定

    ```bash
    sudo systemctl enable docker
    sudo systemctl start docker
    ```

## プログラミング言語

- Python 仮想環境

    ```bash
    python3 -m venv venv
    source venv/bin/activate
    ```

- 開発ツール

    ```bash
    pip install bandit isort ruff
    ```

