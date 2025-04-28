Service の作成
===

1. Pod の起動

    * YAML の作成

        ```yaml title="02_pod_nginx.yaml"
        apiVersion: v1
        kind: Pod
        metadata:
          name: nginx-02
          labels:
            app: Nginx02
        spec:
          containers:
            - image: nginx
              name: nginx
          restartPolicy: Never
        ```

    * Pod の起動

        ```bash
        kubectl apply -f 02_pod_nginx.yaml
        ```

    * Pod の起動確認 (ラベル付き)

        ```bash
        kubectl get pods --show-labels
        ```


2. Service の作成

    * YAML の作成

        ```yaml title="02_service_nginx.yaml"
        apiVersion: v1
        kind: Service
        metadata:
          name: nginx-02-service
        spec:
          selector:
            app: Nginx02
          ports:
          - protocol: TCP
            port: 80
            targetPort: 80
        ```

    * Service の作成

        ```bash
        kubectl apply -f 02_service_nginx.yaml
        ```

    * 作成したサービスの確認

        ```bash
        kubectl get service
        ```

    * エンドポイントと Pod の紐づけ確認

        ```bash
        kubectl get endpoints
        kubectl get pods --selector app=Nginx02 -o wide
        ```

        エンドポイントの IP アドレスと Nginx02 Pod の IP アドレスが同じになっている。

3. ブラウザでの動作確認

    * Port Forward の設定

        ```bash
        nohup kubectl port-forward service/nginx-02-service 8080:80 &
        ```

        上記コマンドでは、

        ```text
        Forwarding from 127.0.0.1:8080 -> 80
        Forwarding from [::1]:8080 -> 80
        ```

        となるため、localhost 以外からはアクセスすると `Connection refused` になります。  
        localhost 以外からアクセスする場合は下記の例のように、`--address` でバインドする IP アドレスを指定します。

        ```bash
        nohup kubectl port-forward service/nginx-02-service 8080:80 --address=0.0.0.0 &
        ```

4. Nginx Pod のログ確認

    ```bash
    kubectl logs nginx-02 -f
    ```

    `-f` は強制を意味するのではなく `--follow` (tail コマンドと感じ) の意味になります。

5. 作成したリソースの削除

    ```bash
    kubectl delete -f .
    ```

    ファイル名のところにピリオドを指定すると、カレントディレクトリに格納している YAML ファイルから作成したリソースを、順に削除してくれます。
