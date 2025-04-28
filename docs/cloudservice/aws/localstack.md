[LocalStack](https://www.localstack.cloud/) 使い方メモ
===

## LocalStack とは

AWS の CLI / API ベースのエミュレータです。マネージメントコンソールはないっぽい。  
ライセンス料を支払うとエミュレートしてくれる機能が増え、ともすると、AWS 環境無しに AWS サービスを使用したアプリを作成することもできるようです。

## [インストール](https://docs.localstack.cloud/getting-started/installation/)

公式の Docker コンテナが用意されているので、それを使うと、簡単に環境構築を行えます。

* docker-compose.yaml

    ```yaml
    version: "3.8"

    services:
      # LocalStack
      localstack:
        container_name: localstack
        image: localstack/localstack
        ports:
        - "127.0.0.1:4566:4566"            
        - "127.0.0.1:4510-4559:4510-4559"
        environment:
        - DOCKER_HOST=unix:///var/run/docker.sock
        volumes:
        - "/var/run/docker.sock:/var/run/docker.sock"

      # LocalStack 操作用の Linux クライアント
      localstack_client:
        build: ./_ubuntu
        container_name: localstack_client
        hostname: localstack_client
        restart: always
        #volumes:
        #  - ./volume/home:/home
        tty: true
        #command: >
        #  sh -c "bash"
        environment:
          TZ: Asia/Tokyo
          ENDPOINT_URL: http://localstack:4566
    ```

AWS CLI/SDK がインストールされたクライアント端末を用意しておきます。

* ./_ubuntu/Dockerfile

    ```dockerfile
    # Base Image
    FROM ubuntu:22.04

    # 必要なパッケージのインストール
    RUN apt-get update && \
        DEBIAN_FRONTEND=noninteractive apt-get install -y \
            locales tzdata sudo vim git \
            python3 python3-pip \
        && apt-get autoremove -y \
        && apt-get autoclean -y \
        && apt-get clean -y

    # localesの設定
    RUN localedef -f UTF-8 -i ja_JP ja_JP.UTF-8
    ENV LANG="ja_JP.UTF-8" \
        LANGUAGE="ja_JP:ja" \
        LC_ALL="ja_JP.UTF-8"

    # AWSCLIv2
    WORKDIR /opt
    RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" && \
        unzip awscliv2.zip && \
        ./aws/install && \
        rm -rf aws awscliv2.zip
    RUN apt-get install -y groff

    # AWS SDK
    RUN pip3 install boto3

    RUN useradd -u 1000 -g 100 -G sudo -s /bin/bash -d /home/ubuntu ubuntu && \
        echo "ubuntu:ubuntu" | chpasswd

    USER ubuntu
    WORKDIR /home/ubuntu

    CMD ["bash"]
    ```

* コンテナの実行

    ```bash
    docker compose up -d --build
    ```


## 利用方法

お試し用の `localstack_client` コンテナに入って作業します。  
フリー版だと永続データを保持できない制約があるようですが、考えようによっては、コンテナを再起動すれば初期状態に戻せるので、動作確認用途としては便利なところもあります。  

```bash
docker exec -it localstack_client bash
```

### エミュレートできるサービスの確認

```bash
curl http://localstack:4566/_localstack/health | jq

{
  "services": {
    "acm": "available",
    "apigateway": "available",
    "cloudformation": "available",
    "cloudwatch": "available",
    "config": "available",
    "dynamodb": "available",
    "dynamodbstreams": "available",
    "ec2": "running",
    "es": "available",
    "events": "available",
    "firehose": "available",
    "iam": "available",
    "kinesis": "available",
    "kms": "available",
    "lambda": "available",
    "logs": "available",
    "opensearch": "available",
    "redshift": "available",
    "resource-groups": "available",
    "resourcegroupstaggingapi": "available",
    "route53": "available",
    "route53resolver": "available",
    "s3": "available",
    "s3control": "available",
    "scheduler": "available",
    "secretsmanager": "available",
    "ses": "available",
    "sns": "available",
    "sqs": "available",
    "ssm": "available",
    "stepfunctions": "available",
    "sts": "available",
    "support": "available",
    "swf": "available",
    "transcribe": "available"
  },
  "edition": "community",
  "version": "3.1.1.dev"
}
```

### AWS CLI

* AWS CLI のサブコマンド tab 補完機能の有効化

    ```bash
    echo "complete -C '`which aws_completer`' aws" >> $HOME/.bashrc
    source $HOME/.bashrc
    ```

* AWS CLI の初期化

    ```bash
    aws configure --profile localstack

    AWS Access Key ID [None]: dummy
    AWS Secret Access Key [None]: dummy
    Default region name [None]: ap-northeast-1
    Default output format [None]: json
    ```

    Access Key、Secret Access Key は適当でよいのですが、Region は AWS の正しいリージョンコードを設定する必要があるようです。

* 動作確認

    ```bash
    aws ec2 describe-instances \
        --profile localstack \
        --endpoint-url $ENDPOINT_URL
    ```

    環境変数 `ENDPOINT_URL` は docker-compose.yml の `environment` で定義しています。


* IAM 関連コマンド

    * ユーザ追加

        ```bash
        aws iam create-user \
            --user-name testuser \
            --profile localstack \
            --endpoint-url $ENDPOINT_URL
        ```

    * アクセスキーの生成

        ```bash
        aws iam create-access-key \
            --user-name testuser \
            --profile localstack \
            --endpoint-url $ENDPOINT_URL
        ```

    * ユーザリストの取得

        ```bash
        aws iam list-users \
            --profile localstack \
            --endpoint-url $ENDPOINT_URL
        ```

    * 個別ユーザ情報取得

        ```bash
        aws iam get-user \
            --user-name testuser \
            --profile localstack \
            --endpoint-url $ENDPOINT_URL
        ```


### [AWS SDK](https://aws.amazon.com/jp/developer/tools/)

* Boto 3 (AWS SDK for Python)

    * サンプルコード

        ```python
        import boto3

        ACCESS_KEY = "dummy"
        SECRET_ACCESS_KEY = "dummy"
        REGION_NAME = "ap-northeast-1"
        ENDPOINT_URL = "http://localstack:4566"

        ec2 = boto3.client(
                "ec2",
                aws_access_key_id=ACCESS_KEY,
                aws_secret_access_key=SECRET_ACCESS_KEY,
                region_name=REGION_NAME,
                endpoint_url=ENDPOINT_URL
            )

        ec2_instances = ec2.describe_instances()
        print(ec2_instances)
        ```

        Access Key、Secret Access Key は適当でよいのですが、Region は AWS の正しいリージョンコードを設定する必要があるようです。
        
        * [boto3.client の引数](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/core/session.html#boto3.session.Session.client)

