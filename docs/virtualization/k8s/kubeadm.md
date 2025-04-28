[kubeadm](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
===

!!! note
    * 1 台あたり 2 GB以上のメモリ
    * 2 コア以上の CPU
    * クラスター内のすべてのマシン間で通信可能なネットワーク
    * ユニークな hostname、MACアドレス、と product_uuid  
    物理マシンの場合は気にする必要はないですが、仮想マシンの場合は気を付けましょう。
    * マシン内の[特定のポート](https://kubernetes.io/ja/docs/reference/networking/ports-and-protocols/)が開いていること。
    * Swapがオフであること。kubeletが正常に動作するためにはswapは必ずオフでなければなりません。
    ```bash
    sudo sed -i -r 's/(.*swap.img)/#\1/g' /etc/fstab
    ```
    のような感じで、swap パーティション／swap ファイルのマウント設定を解除して再起動すると Swap をオフにできます。


## kubeadm、kubelet、kubectlのインストール

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

## ネットワーク設定

```bash
echo "net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-arptables = 1" | \
    sudo tee /etc/sysctl.d/99-bridge-nf-call-iptables
sudo sysctl -p /etc/sysctl.d/99-bridge-nf-call-iptables 
```


## 初期設定

1. コントロールプレーンノードの初期化

    コントロールプレーンノードの初期化は `kubeadm init` で実行します。  
    また、何らかの理由でコントロールプレーンをリセットしたい場合は、`sudo kubeadm reset` ⇒ `sudo kubeadm init` のようにリセットコマンドを実行してください。

    ```bash
    sudo kubeadm init --pod-network-cidr=192.168.0.0/16
    ```

    ネットワークプラグインに Calico を使用するため、 `-pod-network-cidr` オプションで CIDR を指定します。  
    初期化コマンドの最後に表示される `kubeadm join` コマンドは、このコントロールプレーンノードの K8s クラスタにノードを追加する時に必要になるので控えておきます。  

    このコマンドを実行して下記のようなエラーが出る場合は containerd を疑ってください。

    ```bash
    $ sudo kubeadm init
    [init] Using Kubernetes version: v1.28.2
    [preflight] Running pre-flight checks
    error execution phase preflight: [preflight] Some fatal errors occurred:
            [ERROR CRI]: container runtime is not running: output: time="2023-10-03T12:48:44Z" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
    , error: exit status 1
    [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
    To see the stack trace of this error execute with --v=5 or higher
    ```

    * containerd が古い場合

        containerd ⇒ containerd.io に更新します。

        ```bash
        sudo apt remove containerd
        sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
        sudo apt install containerd.io
        sudo systemctl restart containerd
        ```

    * 元々 containerd.io だった場合

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

    [Quickstart for Calico on Kubernetes | Calico Documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

    * インストール

        ```bash
        kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
        kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
        ```

    * `watch kubectl get pods -n calico-system` を実行して、`STATUS` が `Running` になるまでしばらく待つ。 概ね 5 分くらい。

        ```bash
        NAME                                       READY   STATUS    RESTARTS      AGE
        calico-kube-controllers-5658c687c5-5wfj8   1/1     Running   0             5m14s
        calico-node-86jw5                          0/1     Running   3 (32s ago)   5m18s
        calico-typha-6446877f87-dddlv              1/1     Running   0             5m21s
        csi-node-driver-7rl2w                      2/2     Running   0             5m17s
        ```

    * 自ノードの状態確認

        `kubectl get nodes -o wide` コマンドで自ホスト名の STATUS が Ready になっていれば Calico が正しくインストールできています。

        ```bash
        NAME               STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
        <your-hostname>    Ready    control-plane   14m   v1.28.2   10.0.2.15     <none>        Ubuntu 20.04.6 LTS   5.4.0-163-generic   containerd://1.6.24
        ```

4. Control Plane ノードで Pod を起動できるようにする

    セキュリティ上の理由で、Control Plane ノードでは Pod を起動しない方がよいらしい。

    ```bash
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

    * pod の起動

        ```bash
        kubectl run docker-2048 --image=alexwhen/docker-2048 --port=80
        kubectl get pod
        ```

    * サービスの起動

        ```bash
        kubectl run docker-2048 --image=alexwhen/docker-2048 --port=80
        kubectl get service
        ```

    ブラウザで `http://localhost:30829/` を参照すると 2048 を楽しめると思います。


5. K8s にワーカーノードを追加する

    ```bash
    sudo kubeadm join 10.0.2.15:6443 --token sa25p1.iha95ow1vmn9q6gy \
        --discovery-token-ca-cert-hash sha256:8fe1116f607aca35238bd76c80b73e56c386b996d2039fb41b61490d79332758
    ```

    token と ハッシュ値が分からなくなった場合は、コントロールプレーンノードで下記コマンドを実行してください。

    * token の値が分からなくなった場合

        ```bash
        kubeadm token list
        ```

    * token を再発行する場合

        ```bash
        kubeadm token create
        ```

    * `--discovery-token-ca-cert-hash` のハッシュ値が分からない場合

        ```bash
        openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
            openssl rsa -pubin -outform der 2>/dev/null | \
            openssl dgst -sha256 -hex | sed 's/^.* //'
        ```

## 参考サイト

* [2019年版・Kubernetesクラスタ構築入門](https://knowledge.sakura.ad.jp/20955/)
* [kubeadmを使用したクラスターの作成](https://kubernetes.io/ja/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
* [ローカルホストでのKuberntes 環境構築手順](https://www.scsk.jp/lib/product/oss/pdf/oss_27.pdf)
