GitLab Container
===

## 環境構築

下記のようなファイル構成を想定しています。

!!! important
    gitlab_container  
    |-- certs/  
    |-- docker-compose.yaml

### サーバ証明書の用意

GitLab コンテナ構築時にサーバ証明書も用意してくれるのですが、有効期限が 1 カ月と短く、gitlab-runner を使うときに支障が出るので、自前のサーバ証明書を用意します。  
秘密鍵はパスワードを設定するか、SoftHSM のようなアプリで管理すべきだと思いますが、自分しか使用者がいないので、その辺は考えないことにしています。  
以下の作業は `gitlab_container/certs` で実行することを前提にしています。

- 自己認証局秘密鍵・公開鍵の作成

    ```bash
    openssl req \
        -new \
        -out privateca.crt \
        -keyout privateca.key \
        -x509 \
        -days 3650 \
        -newkey rsa:2048 \
        -nodes \
        -subj "/CN=PrivateCA"
    ```

- サーバ証明書作成

    - CSR

        ```bash
        openssl req \
            -new \
            -out dev.gitlab.local.csr \
            -keyout dev.gitlab.local.key \
            -newkey rsa:2048 \
            -nodes \
            -subj "/CN=dev.gitlab.local"
        ```
     
    - Subject Alternative Name の準備

        ここでは `san.txt` とします。

        ```text
        echo "subjectAltName = DNS:dev.gitlab.local,DNS:localhost,IP:127.0.0.1,IP:192.168.1.100" \
            > san.txt
        ```
     
    - CSR へ署名

        ```bash
        openssl x509 \
            -req \
            -days 3650 \
            -in dev.gitlab.local.csr \
            -out dev.gitlab.local.crt \
            -CA privateca.crt \
            -CAkey privateca.key \
            -CAcreateserial \
            -extfile san.txt
        ```

### コンテナの用意

- docker-compose.yaml

```yaml
version: '3'
services:
  gitlab:
    image: 'gitlab/gitlab-ce:17.5.1-ce.0'
    container_name: gitlab
    restart: always
    hostname: gitlab
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        # Add any other gitlab.rb configuration here, each on its own line
        external_url 'https://dev.gitlab.local'
    ports:
      - '80:80'
      - '443:443'
      - '22:22'
    volumes:
      - './volume/config:/etc/gitlab'
      - './volume/logs:/var/log/gitlab'
      - './volume/data:/var/opt/gitlab'
      - './certs:/etc/gitlab/ssl'
    shm_size: '256m'

  #gitlab-runner:
  #  image: gitlab/gitlab-runner:latest
  #  restart: always
  #  volumes:
  #    - ./volume/gitlab-runner/config:/etc/gitlab-runner
  #    - /var/run/docker.sock:/var/run/docker.sock
```

- [Install GitLab using Docker](https://docs.gitlab.com/ee/install/docker.html)
- [docker-composeでGitLab](https://qiita.com/kujiraza/items/f87d2a9fb42ff227d3e6)


## 初回起動

1. STATUS が `health: starting` ⇒ `healthy` になるまで待つ (5 分くらい？)

    ```text
    CONTAINER ID   IMAGE                     COMMAND             CREATED         STATUS                         
    dc18d732b698   gitlab/gitlab-ce:latest   "/assets/wrapper"   4 minutes ago   Up 9 seconds (health: starting)
    ```
    ↓
    ```text
    CONTAINER ID   IMAGE                     COMMAND             CREATED         STATUS                         
    dc18d732b698   gitlab/gitlab-ce:latest   "/assets/wrapper"   19 minutes ago   Up 15 minutes (healthy)
    ```

2. 初回ログイン

    デフォルトユーザ名は `root` で、デフォルトパスワードは下記コマンドで取得できます。  
    initial_root_password は暫くすると自動で削除されるので、早めに root パスワードを変更するか、このファイルのコピーを保存しておきましょう。

    ```bash
    docker exec gitlab cat /etc/gitlab/initial_root_password
    ```

    ![](./gitlab/00_gitlab_login.png)
