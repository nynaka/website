Pod の起動
===

## YAML ファイルを使った起動 (推奨される方法)

1. yaml ファイルの準備

    ```yaml: 01_pod_nginx.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx-01
    spec:
      containers:
        - image: nginx
          name: nginx
    ```

2. Nginx Pod の起動

    ```bash
    kubectl apply -f 01_pod_nginx.yaml
    ```

3. Pod の起動確認

    ```bash
    kubectl get pods
    ```

4. Pod の設定 (未指定項目のデフォルト値含む) および状態確認

    ```bash
    kubectl get pod nginx-01 -o yaml
    ```

5. Pod の削除

    ```bash
    kubectl delete -f 01_pod_nginx.yaml
    ```


## YAML ファイルを使った起動 (非推奨の方法)

1. Pod の起動

    ```bash
    kubectl run nginx --image=nginx
    ```

2. Pod の起動確認

    ```bash
    kubectl get pods
    ```

3. Pod の設定 (未指定項目のデフォルト値含む) および状態確認

    ```bash
    kubectl get pod nginx -o yaml
    ```

4. Pod の削除

    ```bash
    kubectl delete pod nginx
    ```


## Pod 定義用の yaml ファイルの作り方

```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > nginx-pod.yaml
```
