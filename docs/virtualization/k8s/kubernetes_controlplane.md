コントロールプレーンの初期化
===

## kubeadm の初期化

コントロールプレーンノードの初期化は `kubeadm init` で実行します。  
また、何らかの理由でコントロールプレーンをリセットしたい場合は、`sudo kubeadm reset` ⇒ `sudo kubeadm init` のようにリセットコマンドを実行してください。

```bash
$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16
[init] Using Kubernetes version: v1.28.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster

・・・

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.62.130:6443 --token 153nr1.z3bvctgay3myz86h \
        --discovery-token-ca-cert-hash sha256:c237ca998bca7ca18116c0d78c366d7ee0f3edd9a396550b3d8b8e8f2241a277
```

初期化コマンドの最後に表示される `kubeadm join` コマンドは、このコントロールプレーンノードの K8s クラスタにノードを追加する時に必要になります。  
このコマンドを確認するコマンドもありますので、それほど重要な情報ではありません。


* よくある？問題

    * containerd 関連のエラーが出る

        ```bash
        $ sudo kubeadm init --pod-network-cidr=192.168.0.0/16
        [init] Using Kubernetes version: v1.28.2
        [preflight] Running pre-flight checks
        error execution phase preflight: [preflight] Some fatal errors occurred:
                [ERROR CRI]: container runtime is not running: output: time="2023-10-12T12:27:45Z" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
        , error: exit status 1
        [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
        To see the stack trace of this error execute with --v=5 or higher
        ```

    * 対処方法

        * その 1

            containerd ⇒ containerd.io に更新します。

            ```bash
            sudo apt remove containerd
            sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
            sudo apt install containerd.io
            sudo systemctl restart containerd
            ```

        * その 2

            元々 containerd.io だった場合は config.toml を無効にして再起動します。

            ```bash
            sudo mv /etc/containerd/config.toml /etc/containerd/config.toml.bak
            sudo systemctl restart containerd
            ```

        config.toml が原因だった場合、config.toml を元に戻して containerd を再起動すれば同じエラーが再現するようになるはずですが、一度、エラーが発生しない状態になると、config.toml を戻しても、それまでのエラーは発生しなくなるようです。


## kubeadm のコンフィグ設定

設定ファイルをコピーしてパーミッションを設定するだけです。

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


### kubectl コマンドの補完

* bash

    ```bash
    kubectl completion bash > $HOME/.bashrc_k8s
    echo "source $HOME/.bashrc_k8s" >> $HOME/.bashrc
    source $HOME/.bashrc
    ```

    `kubectl [tabキー]` で、kubectl のサブコマンドの候補が表示されれば正しく設定ができています。


## Calico ネットワークプラグインのインストール

[Quickstart for Calico on Kubernetes | Calico Documentation](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)

* インストール

    ```bash
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
    kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
    ```

* `watch kubectl get pods -n calico-system` を実行して、`STATUS` が `Running` になるまでしばらく待つ。   
  概ね 3 分くらい。

    ```text
    Every 2.0s: kubectl get pods -n calico-system                                            k8s01: Thu Oct 12 12:57:56 2023

    NAME                                       READY   STATUS    RESTARTS   AGE
    calico-kube-controllers-68d97999db-96w69   1/1     Running   0          2m28s
    calico-node-6w4d7                          1/1     Running   0          2m28s
    calico-typha-d74584495-k76dc               1/1     Running   0          2m29s
    csi-node-driver-zhffl                      2/2     Running   0          2m28s
    ```

* 自ノードの状態確認

    ```bash
    $ kubectl get nodes -o wide
    NAME    STATUS   ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
    k8s01   Ready    control-plane   23m   v1.28.2   192.168.62.130   <none>        Ubuntu 20.04.6 LTS   5.4.0-164-generic   containerd://1.6.24
    ```
