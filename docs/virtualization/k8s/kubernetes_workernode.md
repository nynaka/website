Worker Node の追加
===

Control Plane に Worker Node を追加する時は、Control Plane の kubeadm 初期化時に表示される `kubeadm join` コマンドを実行します。  
`kubeadm join` コマンドは、Control Plane で `kubeadm token create --print-join-command` を実行することで取得できます。

* Worker Node で実行するコマンド 
```bash
sudo kubeadm join 192.168.62.130:6443 --token ls7sv3.cv06emhjbhszz25j \
        --discovery-token-ca-cert-hash sha256:9c406cf362c6abc9c1331ffbc9b62cf134957259752a445529d4bf2be91d5bf5
```

## token とハッシュ値の確認方法

token とハッシュ値は下記の方法で調べる／作成することができます。

* token の値が分からなくなった場合

    コントロールプレーンで下記コマンドを実行すると取得できます。

    ```bash
    kubeadm token list
    ```

* token を再発行する場合

    token は、コントロールプレーンを再起動したりした場合に、再発行する必要があります。

    ```bash
    kubeadm token create
    ```

* `--discovery-token-ca-cert-hash` のハッシュ値が分からない場合

    コントロールプレーンで下記コマンドを実行すると取得できます。

    ```bash
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | \
        openssl rsa -pubin -outform der 2>/dev/null | \
        openssl dgst -sha256 -hex | sed 's/^.* //'
    ```


### よくある？問題

* containerd 関連のエラーが出る

    ```bash
    $ sudo kubeadm join 172.16.1.100:6443 --token 99u2ik.uau4zk9ah6gvl7yf --discovery-token-ca-cert-hash sha256:720117193bfa60510bc177cb5d98d2d48668a12c239040a4b32109f6f0cbdaf0

    [preflight] Running pre-flight checks
    error execution phase preflight: [preflight] Some fatal errors occurred:
            [ERROR CRI]: container runtime is not running: output: time="2023-10-15T00:22:32Z" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
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


## Worker Node 追加の確認

Control Plane で `kubectl get nodes` を実行すると、Worker Node の追加状況を確認できます。

```bash
$ kubectl get nodes
NAME    STATUS   ROLES           AGE     VERSION
k8s01   Ready    control-plane   25m     v1.28.2
k8s02   Ready    <none>          2m29s   v1.28.2    <<< これが追加した Worker Node
```

!!! note

    Worker Node では kubectl コマンドは使用しないため、コンフィグファイルのコピーは不要です。


## 動作確認

!!! note

    Control Plane で実行します。

1. Pod の起動

    * pod.yaml の作成

        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: app1
          labels:
            app: App1
        spec:
          containers:
            - name: nginx
              image: nginx
          restartPolicy: Never
        ```

    * Pod の起動

        ```bash
        kubectl apply -f pod.yaml
        ```

    * Pod の動作確認

        ```bash
        $ kubectl get pod
        NAME   READY   STATUS    RESTARTS   AGE
        app1   1/1     Running   0          2m12s
        ```

        STATUS が Running であれば正常に動作しています。  
        `kubectl describe pod app1` の `Node` を確認すると、どの Worker Node で Pod が動作しているのか確認できます。

2. Service の登録

    * service.yaml の作成

        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: app1
        spec:
          selector:
            app: App1
          ports:
          - protocol: TCP
            port: 80
            targetPort: 80
        ```

    * Service の登録

        ```bash
        kubectl apply -f service.yaml
        ```

3. ブラウザでの動作確認

`kubectl get endpoints` コマンドで登録したサービスにアクセスするエンドポイント URL を取得し、取得した URL をブラウザで参照します。  
Nginx のデフォルト画面が参照できれば、正しく動作しています。

```bash
$ kubectl get endpoints
NAME         ENDPOINTS            AGE
app1         192.168.236.130:80   3m28s    <<< 192.168.236.130:80 をブラウザで参照する
kubernetes   172.16.1.100:6443    44m
```

## その他

* 特定の Worker Node を指定して Pod を起動する

    複数の Worker Node で運用する場合、特定の Pod だけは Worker Node を指定して起動したい場合があります。その場合は、Pod 起動時の `nodeName` パラメータで、Worker Node のホスト名を指定してください。

    * 設定例

        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: app1
          labels:
            app: App1
        spec:
          nodeName: k8s02    # Pod を起動する Worker Node
          containers:
            - name: nginx
              image: nginx
          restartPolicy: Never
        ```

        参照サイト: [特定のノードにスケジューリングされるPodを作成する](https://kubernetes.io/ja/docs/tasks/configure-pod-container/assign-pods-nodes/#create-a-pod-that-gets-scheduled-to-specific-node)
