Containerd
===

!!! note

    Docker ⇒ Containerd へ移行中のメモです。nerdctl コマンドを実行するユーザの権限を調査中です。


## インストール

### Containerd

1. Download containerd

    ```bash
    wget https://github.com/containerd/containerd/releases/download/v1.7.7/containerd-1.7.7-linux-amd64.tar.gz
    sudo tar zxvf containerd-1.7.7-linux-amd64.tar.gz -C /usr/local/
    ```

2. Configure containerd

    ```bash
    sudo wget \
        https://raw.githubusercontent.com/containerd/containerd/main/containerd.service \
        -O /etc/systemd/system/containerd.service

    sudo systemctl daemon-reload
    sudo systemctl start containerd
    sudo systemctl enable containerd
    ```


### nerdctl

1. 関連ツールのインストール

    ```bash
    sudo apt install -y rootlesskit
    ```

2. nerdctl のインストール

    ```bash
    wget https://github.com/containerd/nerdctl/releases/download/v1.6.2/nerdctl-1.6.2-linux-amd64.tar.gz
    sudo tar zxvf nerdctl-1.6.2-linux-amd64.tar.gz -C /usr/local/bin/
    containerd-rootless-setuptool.sh install
    ```

3. BuildKit の有効化

    * BuildKit のインストール

        ```bash
        wget https://github.com/moby/buildkit/releases/download/v0.12.3/buildkit-v0.12.3.linux-amd64.tar.gz
        sudo mkdir /usr/local/buildkit
        sudo tar zxvf buildkit-v0.12.3.linux-amd64.tar.gz -C /usr/local/buildkit

        echo "export PATH=$PATH:/usr/local/buildkit/bin" >> $HOME/.bashrc
        source $HOME/.bashrc
        ```

    * BuildKit の有効化

        ```bash
        CONTAINERD_NAMESPACE=default containerd-rootless-setuptool.sh install-buildkit-containerd
        ```

4. ranc のインストール

    ```bash
    sudo wget https://github.com/opencontainers/runc/releases/download/v1.1.9/runc.amd64 -O /usr/local/bin/runc
    sudo chmod +x /usr/local/bin/runc
    ```

5. CNI Plugin のインストール

    ```bash
    wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
    sudo mkdir -p /opt/cni/bin
    sudo tar -zxvf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
    ```

6. 動作確認

    * nerdctl コマンド

        ```bash
        sudo nerdctl run -d -p 8080:80 --name nginx-test nginx
        ```

    * nerdctl compose コマンド

        * docker-compose.yaml

            ```yaml
            version: "3"
            services:
              nginx:
                image: nginx:latest
                restart: always
                ports:
                  - "8080:80"
            ```

        * コンテナの起動

            ```bash
            sudo nerdctl compose up -d --build
            ```

    ブラウザや curl コマンド等で `http://localhost:8080/` を参照すると nginx のデフォルト画面が参照できると思います。  

    コマンドオプションを見る限りでは docker コマンド ⇒ nerdctl コマンドは、実行するコマンドを変更する程度で移行できそうです。`docker compose` コマンドについても `nerdctl compose` で同じ YAML ファイルが使用できそうです。


## 参考サイト

* containerd
    * [containerdとは](https://www.designet.co.jp/faq/term/?id=Y29udGFpbmVyZA)
    * [Containerd, Docker, Kubernetesの関係について](https://forum.ficusonline.com/t/topic/458)

* nerdctl
    * [containerdを使用したコンテナ開発](https://tech-lab.sios.jp/archives/28641)
    * [Containerdとnerdctlで個人検証環境のDockerを置き換えた](https://zenn.dev/igeta/articles/1507f1c3311814)
    * [Dockerからcontainerdへの移行 (NTT Tech Conference 2022発表レポート)](https://medium.com/nttlabs/docker-to-containerd-4f3a56e6f2b6)
