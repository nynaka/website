Namespace の作成
===

!!! note

    Namespace は Kubernetes のリソースを論理的に分離して使用する仕組みです。

## 初期 Namespace

* default

    デフォルトのNamespace。リソース作成時に Namespace を指定しなかった場合は、default Namespace に作成される。

* kube-system

    Kubernetes のシステムによって作成されたオブジェクトのための Namespace。

    * kube-proxy、coredns 等が作成される。
    * metrics-server、kube-state-metrics、node-exporter などのクラスタの状態を管理する。

* kube-public

    認証されていないユーザを含む、すべてのユーザが読み取り可能

* kube-node-lease

    Lease というオブジェクトのための Namespace

## Namespace を分けて管理するメリット

* Pod やコンテナのリソース (メモリ、CPU、ストレージ) 使用上限が設定可能
* Namespace 全体の総リソース (メモリ、CPU、ストレージおよび Pod の数や Object の数) の制限が可能
* 権限設定を他の Namespace から分離できる


## Namespace の操作

1. Namespace の作成 (YAML で作成する場合)

    * YAML ファイルの作成

        ```yaml: 03-namespace.yaml
        apiVersion: v1
        kind: Namespace
        metadata:
          name: testns1
        ```

    * Namespace の作成

        ```bash
        kubectl apply -f 03-namespace.yaml
        ```

    * Namespace の確認

        ```bash
        kubectl get namespaces
        ```

2. Namespace の作成 (kubectl で直接作成する場合)

    ```bash
    kubectl create namespace testns
    ```


## 作成した Namespace で Pod を起動する

### YAML ファイルを利用する場合

* Namespace の作成

    ```bash
    cat << EOF > 03-namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: testns1
    EOF
    ```

    ```bash
    kubectl apply -f 03-namespace.yaml
    ```

* Pod の起動

    ```bash
    cat << EOF > 03-pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-03
      namespace: testns1
    spec:
      containers:
      - image: nginx
        name: nginx
      restartPolicy: Never
    EOF
    ```

    ```bash
    kubectl apply -f 03-pod.yaml
    ```

* Pod の動作確認

    ```bash
    kubectl get pod --namespace testns1
    ```

* Namespace の削除

    ```bash
    kubectl delete -f 03-namespace.yaml
    ```

    Namespace を削除すると、Namespace に紐づくリソースも削除されます。


### kubectl コマンドの場合

* Namespace の作成

    ```bash
    kubectl create namespace testns2
    ```

* Pod の起動

    ```
    kubectl run nginx --image nginx -n testns2
    ```

* Pod の動作確認

    ```bash
    kubectl get pod --namespace testns2
    ```

* Namespace の削除

    ```bash
    kubectl delete namespace testns2
    ```
