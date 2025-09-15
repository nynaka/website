Docker
===

## インストール

### [Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

1. 古いバージョンの削除 (インストールしていた場合)

    ```bash
    sudo apt-get remove docker docker-engine docker.io containerd runc
    ```

2. APT リポジトリの設定

    - 関連ツールのインストール

        ```bash
        sudo apt-get update
        sudo apt-get -y install \
            ca-certificates \
            curl
        ```

    - GPG key のインストール

        ```bash
        sudo install -m 0755 -d /etc/apt/keyrings
        sudo curl \
            -fsSL https://download.docker.com/linux/ubuntu/gpg \
            -o /etc/apt/keyrings/docker.asc
        sudo chmod a+r /etc/apt/keyrings/docker.asc
        ```

    - リポジトリの追加

        ```bash
        echo \
            "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
            $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
            sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
        ```

3. Docker のインストール

    ```bash
    sudo apt-get update
    sudo apt-get install -y \
        docker-ce \
        docker-ce-cli \
        containerd.io \
        docker-buildx-plugin \
        docker-compose-plugin
    ```

4. ユーザ権限の設定

    ```bash
    sudo usermod -aG docker xubuntu
    ```

5. 自動起動設定

    ```bash
    sudo systemctl enable docker
    sudo systemctl start docker
    ```


## Dockerfile サンプル

### Ubuntu コンテナ

#### 特権コンテナ

- Dockerfile

    ```dockerfile
    # Base Image
    FROM ubuntu:24.04

    # systemd のインストール
    RUN apt-get update && \
        DEBIAN_FRONTEND=noninteractive \
        apt-get install -y init systemd

    #RUN useradd -u 1000 -g 100 -G sudo -s /bin/bash -d /home/ubuntu ubuntu \
    #    && echo "ubuntu:ubuntu" | chpasswd
    #RUN echo "ubuntu:ubuntu" | chpasswd

    # 特権コンテナはroot権限で終わらないと起動時にpermittionエラーになる。
    # Dockerfile内でユーザを切り替えた場合はrootに戻す。
    #USER root

    CMD ["/bin/init"]
    ```

- コンテナのビルドと起動

    - docker コマンドで実行する場合

        - コンテナのビルド

            ```bash
            docker build -t ubuntu:custom .
            ```

        - コンテナの起動

            ```bash
            docker run -it \
                --name ubuntu \
                --hostname ubuntu \
                --restart always \
                --privileged \
                -e TZ=Asia/Tokyo \
                ubuntu:custom \
                bash -c "/sbin/init"
            ```

        - コンテナの停止

            ```bash
            docker stop ubuntu
            ```

    - docker compose

        - docker-compose.yaml

            ```yaml
            services:
                ubuntu:
                    build: ./
                    container_name: ubuntu
                    hostname: ubuntu
                    restart: always
                    privileged: true
                    command: >
                        bash -c "/sbin/init"
                    environment:
                        TZ: Asia/Tokyo
            ```

        - 起動コマンド

            ```bash
            docker compose up -d --build
            ```




