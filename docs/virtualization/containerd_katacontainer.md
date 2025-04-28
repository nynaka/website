Kata Container -containerd-
===

## Install Kata Containers

1. Download Kata Containers

    ```bash
    wget https://github.com/kata-containers/kata-containers/releases/download/3.2.0/kata-static-3.2.0-amd64.tar.xz
    sudo tar Jxvf kata-static-3.2.0-amd64.tar.xz -C /
    ```

2. PATH の設定

    ```bash
    echo 'export PATH=/opt/kata/bin:$PATH' > $HOME/.bashrc_kata
    echo 'source $HOME/.bashrc_kata' >> $HOME/.bashrc
    source $HOME/.bashrc
    ```

3. インストールの確認

    ```bash
    $ kata-runtime --version
    kata-runtime  : 3.2.0
       commit   : 45687e3251604ccc71b595d37f14253c4584cd5f
       OCI specs: 1.0.2-dev
    ```


## Install containerd

1. Download containerd

    ```bash
    wget https://github.com/containerd/containerd/releases/download/v1.7.7/containerd-1.7.7-linux-amd64.tar.gz
    sudo tar zxvf containerd-1.7.7-linux-amd64.tar.gz -C /usr/local/
    ```

2. Configure containerd

    * systemd

        ```bash
        sudo wget \
            https://raw.githubusercontent.com/containerd/containerd/main/containerd.service \
            -O /etc/systemd/system/containerd.service

        sudo systemctl daemon-reload
        sudo systemctl start containerd
        sudo systemctl enable containerd
        ```

    * /etc/containerd/config.yaml の作成

        /etc/containerd/ が無い場合は、

        ```bash
        mkdir -p /etc/containerd/
        ```

        を作成し、下記の内容で /etc/containerd/config.yaml を作成します。

        ```text
        [plugins]
          [plugins."io.containerd.grpc.v1.cri"]
            [plugins."io.containerd.grpc.v1.cri".containerd]
              default_runtime_name = "kata"
              [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
                [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
                  runtime_type = "io.containerd.kata.v2"
        ```

    * containerd の再起動

        ```bash
        sudo systemctl restart containerd
        ```

## 動作確認

ctr コマンドが参照する PATH に /opt/kata/bin を追加する方法が分からなかったので、/usr/bin/ に containerd-shim-kata-v2 のシンボリックリンクを追加します。

* containerd-shim-kata-v2 のシンボリックリンクの追加

    ```bash
    ln -s /opt/kata/bin/containerd-shim-kata-v2 /usr/bin/
    ```

* Kata Container のカーネルの確認

    ```bash
    image="docker.io/library/busybox:latest"
    sudo ctr image pull "$image"
    sudo ctr run --runtime "io.containerd.kata.v2" --rm -t "$image" test-kata uname -a
    ```


## nerdctl

### インストール

* 関連ツールのインストール

    ```bash
    sudo apt install -y rootlesskit
    ```

* nerdctl のインストール

    ```bash
    wget https://github.com/containerd/nerdctl/releases/download/v1.6.2/nerdctl-1.6.2-linux-amd64.tar.gz
    sudo tar zxvf nerdctl-1.6.2-linux-amd64.tar.gz -C /usr/local/bin/
    containerd-rootless-setuptool.sh install
    ```

* BuildKit の有効化

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

* containerd の再起動

    ```bash
    sudo systemctl restart containerd.service
    ```


### CNI Plugin のインストール

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar -zxvf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
```

### 動作確認

```bash
sudo nerdctl run --runtime io.containerd.kata.v2 -d -p 8080:80 --name nginx-test nginx
```



## 参考サイト

* containerd
    * [containerdとは](https://www.designet.co.jp/faq/term/?id=Y29udGFpbmVyZA)
    * [Containerd, Docker, Kubernetesの関係について](https://forum.ficusonline.com/t/topic/458)

* nerdctl
    * [containerdを使用したコンテナ開発](https://tech-lab.sios.jp/archives/28641)
    * [Containerdとnerdctlで個人検証環境のDockerを置き換えた](https://zenn.dev/igeta/articles/1507f1c3311814)
    * [Dockerからcontainerdへの移行 (NTT Tech Conference 2022発表レポート)](https://medium.com/nttlabs/docker-to-containerd-4f3a56e6f2b6)

* [Kata Container](https://katacontainers.io/)
    * [コンテナランタイムKata Containersの実行検証](https://qiita.com/hogehoge789/items/ceb392a3c16278579b2c)
    * [Install Kata Containers with containerd](https://github.com/kata-containers/kata-containers/blob/main/docs/install/container-manager/containerd/containerd-install.md)
    * [How to use Kata Containers and Containerd](https://github.com/kata-containers/documentation/blob/master/how-to/containerd-kata.md)
