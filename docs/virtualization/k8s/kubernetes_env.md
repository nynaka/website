Kubernetes 環境設定・ツールのインストール
===

## ホスト OS の設定 その 1:

* SWAP の無効化

    `/etc/fstab` を参照すると下記の例のようにファイルシステムのタイプにあたるパラメータ (左から3番目) が `swap` の行があると思いますので、この行を削除、または、先頭に `#` を挿入してコメントに変更し、OS を再起動して、SWAP を無効化します。

    ```text
    /swap.img       none    swap    sw      0       0
    ↓
    #/swap.img       none    swap    sw      0       0
    ```

* IP アドレスの固定

    Control Plane の IP アドレスが変更されると `kubeadm reset` からやり直すことになります。  
    面倒ごとを避けるために、最初から IP アドレスは固定しておくことがおすすめです。


## [Docker](https://docs.docker.com/engine/install/ubuntu/)

* Ubuntu Linux

    1. 古いバージョンの削除 (インストールしていた場合)

        ```bash
        sudo apt-get remove docker docker-engine docker.io containerd runc
        ```

    2. APT リポジトリの設定

        * 関連ツールのインストール

            ```bash
            sudo apt-get update
            sudo apt-get -y install \
                ca-certificates \
                curl \
                gnupg \
                lsb-release
            ```

        * GPG key のインストール

            ```bash
            sudo mkdir -p /etc/apt/keyrings
            curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
                sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
            ```

        * リポジトリの追加

            * Debian Linux

                ```bash
                echo \
                    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
                    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
                    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
                ```

            * Ubuntu Linux

                ```bash
                echo \
                    "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
                    "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
                    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
                ```

        * Docker のインストール

            ```bash
            sudo apt-get update
            sudo apt-get install -y \
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


## ホスト OS の設定 その 2: bridge 関連設定

たぶん Docker をインストール時に L2 関連のカーネルモジュールがインストールされるのだと思います。  
Docker インストール後に下記の設定ができるようになります。

```bash
echo "net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1" | \
    sudo tee /etc/sysctl.d/99-bridge-nf-call-iptables

sudo sysctl -p /etc/sysctl.d/99-bridge-nf-call-iptables 
```


## [kubeadm、kubelet、kubectlのインストール](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#kubeadm-kubelet-kubectl%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB)

* Debian / Ubuntu

    1. Kubernetes の apt リポジトリ利用のためのパッケージ追加

        ```bash
        sudo apt-get update
        sudo apt-get install -y apt-transport-https ca-certificates curl
        ```

    2. リポジトリ公開鍵のダウンロード

        ```bash
        curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | \
            sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
        ```

    3. Kubernetes の apt リポジトリの追加

        ```bash
        echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | \
            sudo tee /etc/apt/sources.list.d/kubernetes.list
        ```

    4. kubelet、kubeadm、kubectlをインストール

        ```bash
        sudo apt-get update
        sudo apt-get install -y kubelet kubeadm kubectl
        sudo apt-mark hold kubelet kubeadm kubectl
        ```

    5. kubectl コマンドの補完

        ```bash
        kubectl completion bash >> $HOME/.bashrc_kuberctl
        echo 'source $HOME/.bashrc_kuberctl' >> $HOME/.bashrc
        source $HOME/.bashrc
        ```

        * `kubectl tabキー` で 「_get_comp_words_by_ref: command not found」 が出る

            Bash の自動補完パッケージが足りないので、追加すれば解消します。  
            自動補完パッケージインストール後、再ログインすると有効になります。

            ```bash
            sudo apt install -y bash-completion
            ```

## Kubernets 環境の初期化

1. コントロールプレーンノードの初期化

    コントロールプレーンノードの初期化は kubeadm init で実行します。  
    また、何らかの理由でコントロールプレーンをリセットしたい場合は、sudo kubeadm reset ⇒ sudo kubeadm init のようにリセットコマンドを実行してください。

    ```bash
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16
    ```

    * よくある問題

        下記のエラーが出た場合は containerd を疑ってください。

        ```bash
        $ sudo kubeadm init --pod-network-cidr=192.168.0.0/16
        [init] Using Kubernetes version: v1.28.3
        [preflight] Running pre-flight checks
        error execution phase preflight: [preflight] Some fatal errors occurred:
                [ERROR CRI]: container runtime is not running: output: time="2023-11-04T01:19:36Z" level=fatal msg="validate service connection: CRI         v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc =         unknown service runtime.v1.RuntimeService"
        , error: exit status 1
        [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
        To see the stack trace of this error execute with --v=5 or higher
        ```

        * 対応方法 をの 1 (containerd が古い場合)

            containerd ⇒ containerd.io に更新します

            ```bash
            sudo apt remove containerd
            sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
            sudo apt install containerd.io
            sudo systemctl restart containerd
            ```

        * 対応方法 をの 2 (元々 containerd.io だった場合)

            ```bash
            sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
            sudo systemctl restart containerd
            ```

            config.toml が原因だった場合、config.toml を元に戻して containerd を再起動すれば同じエラーが再現するようになるはずですが、一度、エラーが発生しない状態になると、config.toml を戻しても、それまでのエラーは発生しなくなるようです。

2. kubeadm の初期設定

    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

3. Calico ネットワークプラグインのインストール

    [Quickstart for Calico on Kubernetes](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

    * インストール

        ```bash
        kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/tigera-operator.yaml
        kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.3/manifests/custom-resources.yaml
        ```

        * よくある問題 (10.0.2.15:6443 の接続に拒否される)

            ```text
            The connection to the server 10.0.2.15:6443 was refused - did you specify the right host or port?
            ```

            cgroup のバージョンの問題でこの結果になる説が有力っぽいですが、解決できませんでした。  
            とりあえず、Ubuntu 22.04 LTS でこの現象が出た場合は、Ubuntu 20.04 LTS にすると解消します。Ubuntu 21.04 あたりで cgroup v1 ⇒ v2 になったことによる影響らしい。。。  

            ちなみに、 Debian Linux 12 の場合は grup の `GRUB_CMDLINE_LINUX_DEFAULT` に `systemd.unified_cgroup_hierarchy=false` を設定して再起動すると解消します。

            * /etc/default/grub

                ```text:/etc/default/grub
                GRUB_DEFAULT=0
                GRUB_TIMEOUT=5
                GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
                GRUB_CMDLINE_LINUX_DEFAULT="systemd.unified_cgroup_hierarchy=false"    # この行修正
                GRUB_CMDLINE_LINUX=""
                ```

            * grup の反映

                ```bash
                sudo update-grub
                sudo reboot
                ```

        * `watch kubectl get pods -n calico-system` を実行して、STATUS が Running になるまでしばらく待つ。 概ね 3 分くらい。

            ```text
            Every 2.0s: kubectl get pods -n calico-system                                              k8s: Sat Nov  4 02:46:13 2023

            NAME                                      READY   STATUS    RESTARTS   AGE
            calico-kube-controllers-77bb6497c-pzqbl   1/1     Running   0          2m4s
            calico-node-97dcw                         1/1     Running   0          2m5s
            calico-typha-7bd48fccb4-tb9cr             1/1     Running   0          2m5s
            csi-node-driver-q2xrt                     2/2     Running   0          2m4s
            ```

4. コントロールプレーンでも Pod を起動したい場合

    1 台の Kubernetes ノードだけで運用しようとすると、Pod の Status が Pending のまま、いつまで待っても起動しない問題が発生します。  

    Kubernetes ではセキュリティ上の観点から、デフォルトでは、コントロールプレーンで Pod は起動しないようになっているそうで、このデフォルト設定を無効にする必要があります。

    ```bash
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

    ※ 2 個個目コマンドが成功したところを見たことがないのですが、解説サイトでは上記2コマンドを実行していました。


## その他

### Docker Registry

インターネットで公開されているコンテナイメージではなく、自分でカスタマイズしたコンテナイメージを利用する場合は、ローカルネットワーク上に Docker Registry のようなサービスを作成しておくと便利です。

> [!NOTE]
> * Docker Registry に登録したコンテナイメージを、ブラウザの削除ボタンから削除できません (ボタンは押下できるのだけれど。。。)

* docker-compose.yml

    ```yaml
    version: "3"
    services:

        frontend:
            #image: konradkleine/docker-registry-frontend:v2
            image:  ekazakov/docker-registry-frontend
            container_name: frontend
            hostname: frontend
            environment:
                ENV_DOCKER_REGISTRY_HOST: backend
                ENV_DOCKER_REGISTRY_PORT: 5000
            ports:
                - 80:80

        backend:
            image: registry:2
            container_name: backend
            hostname: backend
            restart: always
            ports:
                - "5000:5000"
            #volumes:
            #    - ./var.lib.registry:/var/lib/registry
            environment:
                REGISTRY_STORAGE_DELETE_ENABLED: 'True'
    ```

* 起動コマンド

    ```bash
    docker compose up -d --build
    ```
