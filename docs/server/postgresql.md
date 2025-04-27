PostgreSQL
===

## インストール

- Redhat系

    - Fedora 41

        - PostgreSQL リポジトリの追加

            ```bash
            sudo dnf install -y \
                https://download.postgresql.org/pub/repos/yum/reporpms/F-$(rpm -E %fedora)-x86_64/pgdg-fedora-repo-latest.noarch.rpm
            ```

        - システムデフォルトの PostgreSQL 無効化

            ```bash
            sudo dnf -qy module disable postgresql
            ```

        - PostgreSQL のインストール

            ```bash
            sudo dnf install -y postgresql17-server
            ```

        - PostgreSQL DB の初期化

            ```bash
            sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
            ```

        - PostgreSQL の起動と自動起動設定

            ```bash
            sudo systemctl enable postgresql-17
            sudo systemctl start postgresql-17
            ```

    - Alma Linux9 / Rocky Linux9

        - インストール可能な PostgreSQL のバージョン確認

            ```bash
            sudo dnf module list postgresql
            ```

        - 使用するバージョンの有効化

            ```bash
            sudo dnf module enable -y postgresql:16
            ```

        - PostgreSQL のインストール

            ```bash
            sudo dnf install -y postgresql-server
            ```

        - PostgreSQL DB の初期化

            ```bash
            sudo postgresql-setup --initdb
            ```

        - PostgreSQL の起動と自動起動設定

            ```bash
            sudo systemctl enable postgresql
            sudo systemctl start postgresql
            ```


- Debian系
    - Debian Linux 12
    - Ubuntu Linux 22.04

        ```bash
        sudo apt install -y postgresql postgresql-contrib
        ```

- サーバ起動設定

    ```bash
    sudo systemctl start postgresql
    sudo systemctl enable postgresql
    ```

- ソースからインストール

    - ビルドツールのインストール

        **configure** スクリプトがエラー終了しなくなるまで依存関係にあるビルドツールやライブラリをインストールしてください。

        - Debian Linux / Ubuntu Linux

            ```bash
            sudo apt install -y bison flex libicu-dev libreadline-dev libssl-dev
            ```

    - ソースの取得

        [Github](https://github.com/postgres/postgres) でも公開されていますが、ここでは [PostgreSQL: File Browser](https://www.postgresql.org/ftp/source/) から tar ファイルを取得するものとします。

        ```bash
        wget https://ftp.postgresql.org/pub/source/v17.2/postgresql-17.2.tar.bz2
        ```

    - ソースのビルド

        ```bash
        tar jxvf postgresql-17.2.tar.bz2
        cd postgresql-17.2
        ./configure --prefix=/usr/local/pgsql --with-openssl
        make
        sudo make install
        ```

    - **postgres** ユーザ作成

        ```bash
        sudo useradd -d /var/lib/postgres -m -s /bin/bash  postgres
        ```

    - DB の初期化

        ```bash
        sudo -i -u postgres
        /usr/local/pgsql/bin/initdb -D /var/lib/postgres/data
        ```

    - PostgreSQL サーバの起動

        ```bash
        /usr/local/pgsql/bin/pg_ctl -D /var/lib/postgres/data -l logfile start
        ```

    - PostgreSQL に接続

        ```bash
        /usr/local/pgsql/bin/psql -U postgres
        ```

    - 環境変数設定

        ```bash
        echo 'export PATH=/usr/local/pgsql/bin:$PATH' >> $HOME/.bashrc
        source $HOME/.bashrc
        ```


## 設定

### PostgreSQL 外部ホスト接続設定

- /etc/postgresql/14/main/postgresql.conf

    ```diff
    --- ./postgresql.conf   2024-12-29 17:21:28.504696740 +0900
    +++ /etc/postgresql/14/main/postgresql.conf     2024-12-29 17:22:06.655694344 +0900
    @@ -57,7 +57,7 @@
    
     # - Connection Settings -
    
    -#listen_addresses = 'localhost'                # what IP address(es) to listen on;
    +listen_addresses = '*'         # what IP address(es) to listen on;
                                            # comma-separated list of addresses;
                                            # defaults to 'localhost'; use '*' for all
                                            # (change requires restart)
    ```

- pg_hba.conf

    ```diff
    --- /etc/postgresql/14/main/pg_hba.conf 2024-12-29 17:32:12.750079113 +0900
    +++ ./pg_hba.conf       2024-12-29 17:31:48.835118567 +0900
    @@ -95,7 +95,6 @@
     local   all             all                                     peer
     # IPv4 local connections:
     host    all             all             127.0.0.1/32            scram-sha-256
    -host    all             all             0.0.0.0/0               scram-sha-256
     # IPv6 local connections:
     host    all             all             ::1/128                 scram-sha-256
     # Allow replication connections from localhost, by a user with the
    ```

- PostgreSQL サーバ再起動

    ```bash
    sudo systemctl restart postgresql
    ```

- 接続テスト用アカウント・DB 作成

    ```bash
    su - postgres
    psql -c "CREATE ROLE testuser WITH SUPERUSER LOGIN PASSWORD 'testpassword';"
    PGPASSWORD="testpassword" createdb -h localhost -U testuser -l C -T template0 -E utf8 testdb
    ```

## psql コマンド

### PostgreSQL に接続

- 管理ユーザ (postgres)

    ```bash
    sudo -i -u postgres psql
    ```


### psql コマンド操作

- ユーザ関連

    - ユーザ追加

        - パスワード無し

            ```sql
            CREATE USER ユーザ名;
            ```

        - パスワード付き

            ```sql
            CREATE USER test WITH PASSWORD 'passwd';
            ```

        - パスワード変更・削除

            ```sql
            # 変更
            ALTER ROLE test WITH PASSWORD 'p@ssw0rd';
            # 削除
            ALTER ROLE test WITH PASSWORD null;
            ```

        - testユーザーにデータベース作成権限付与

            ```sql
            ALTER ROLE test WITH createdb;
            ```

    - ユーザ削除

        ```sql
        DROP USER ユーザ名;
        ```

    - ユーザ一覧確認

        ```sql
        \du
        ```

- PostgreSQL プロンプトの終了

    ```sql
    \q
    または
    exit
    ```


## プログラムからの利用

### 準備

- PostgreSQL ユーザ、DB 作成

    ```bash
    sudo -i -u postgres psql
    ```

    ```sql
    CREATE USER test WITH PASSWORD 'passwd';
    CREATE DATABASE testdb OWNER test;
    GRANT all ON DATABASE testdb TO test;
    \q
    ```

- テーブル作成

    ```bash
    psql -h localhost -U test testdb
    ```

    ```sql
    CREATE TABLE test (
        id SERIAL PRIMARY KEY,
        col1 VARCHAR(50),
        col2 TEXT
    );
    ```

- 作成したテーブルの確認

    |               |                    |
    | :------------ | ------------------ |
    | \d            | テーブル一覧の表示 |
    | \d テーブル名 | テーブル構造の表示 |

- ダミーデータ登録

    ```sql
    INSERT INTO test (col1, col2) VALUES ('テスト1', 'テスト2');
    ```

    ```sql
    SELECT * FROM test;
    ```

### Go

- サンプルコード

    ```bash
    package main

    import (
            "database/sql"
            "fmt"
            "log"

            _ "github.com/lib/pq" // PostgreSQLドライバをインポート（空の識別子が必要）
    )

    func main() {
        // PostgreSQL接続情報
        connStr := "user=test password=passwd dbname=testdb sslmode=disable"

        // データベース接続
        db, err := sql.Open("postgres", connStr)
        if err != nil {
            log.Fatalf("データベース接続エラー: %v", err)
        }
        defer db.Close()

        // 接続確認
        err = db.Ping()
        if err != nil {
            log.Fatalf("データベースPingエラー: %v", err)
        }
        fmt.Println("PostgreSQLに接続しました！")

        // データ挿入
        _, err = db.Exec(`INSERT INTO test (col1, col2) VALUES ($1, $2)`, "Col1 Value", "Col2 Value")
        if err != nil {
                log.Fatalf("データ挿入エラー: %v", err)
        }
        fmt.Println("データを挿入しました！")

        // データ取得
        rows, err := db.Query(`SELECT * FROM test`)
        if err != nil {
            log.Fatalf("データ取得エラー: %v", err)
        }
        defer rows.Close()

        fmt.Println("データを取得しました：")
        for rows.Next() {
            var id int
            var col1 string
            var col2 string
            err = rows.Scan(&id, &col1, &col2)
            if err != nil {
                log.Fatalf("データスキャンエラー: %v", err)
            }
            fmt.Printf("ID: %d, col1: %s, col2: %s\n", id, col1, col2)
        }

        // エラー確認
        if err = rows.Err(); err != nil {
            log.Fatalf("行エラー: %v", err)
        }
    }
    ```

- 実行方法

    1. プロジェクトディレクトリの初期化

        ```bash
        go mod init go_postgres
        ```

    2. github.com/lib/pq の取得

        ```bash
        go get github.com/lib/pq
        ```

    3. サンプルコードの実行

        - その 1
        
            ```bash
            go run pg_sample.go
            ```

        - その 2

            ```bash
            go build pg_sample.go
            ```

### Python

- ライブラリのインストール

    ```bash
    sudo apt install -y libpq-dev
    ```

    ```bash
    pip3 install SQLAlchemy psycopg2
    ```

- サンプルコード

    ```python
    from sqlalchemy import create_engine, Column, Integer, String, text
    from sqlalchemy.orm import declarative_base, sessionmaker

    username = "test"
    password = "passwd"

    # ベースクラスの作成
    Base = declarative_base()

    # テーブルを表すクラスの定義
    class Test(Base):
        __tablename__ = 'test'
        id = Column(Integer, primary_key=True)
        col1 = Column(String)
        col2 = Column(String)

    if __name__ == "__main__":

        #postgreSQLに接続
        engine=create_engine(f"postgresql://{username}:{password}@localhost:5432/testdb")

        # セッションの作成
        Session = sessionmaker(bind=engine)
        session = Session()

        # データの挿入
        new_record = Test(col1='インサートレコード', col2='インサートデータ')
        session.add(new_record)  # セッションに追加
        session.commit()  # トランザクションを確定

        # 全てのレコードを取得
        results = session.query(Test).all()

        # 結果の出力
        for row in results:
            print(f"id: {row.id}, col1: {row.col1}, col2: {row.col2}")
    ```


### Rust

- プロジェクトの作成

    ```bash
    cargo new rust_postgres
    cd rust_postgres
    ```

- tokio-postgres の依存関係の追加

    Cargo.toml

    ```diff
    --- Cargo.toml.origin   2025-01-13 08:52:01.703582760 +0900
    +++ Cargo.toml  2025-01-13 08:52:26.033746605 +0900
    @@ -6,3 +6,5 @@
     # See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
    
     [dependencies]
    +tokio = { version = "1", features = ["full"] }
    +tokio-postgres = "0.7"
    ```

- サンプルコード

    src/main.rs

    ```rust
    use tokio_postgres::{NoTls, Error};

    #[tokio::main]
    async fn main() -> Result<(), Error> {
        // PostgreSQLの接続情報
        let connection_string = "host=localhost user=test password=passwd dbname=testdb";

        // クライアントと接続タスクを取得
        let (client, connection) = tokio_postgres::connect(connection_string, NoTls).await?;

        // 別スレッドで接続タスクを駆動
        tokio::spawn(async move {
            if let Err(e) = connection.await {
                eprintln!("接続エラー: {}", e);
            }
        });

        // データを挿入
        client
            .execute("INSERT INTO test (col1, col2) VALUES ($1, $2)", &[&"Col1 Value", &"Col2 Value"])
            .await?;
        println!("データを挿入しました！");

        // データを取得
        let rows = client.query("SELECT * FROM test", &[]).await?;
        for row in rows {
            let id: i32 = row.get(0);
            let col1: &str = row.get(1);
            let col2: &str = row.get(1);

            println!("ID: {}, col1: {}, col2: {}", id, col1, col2);
        }

        Ok(())
    }
    ```

- サンプルコードの実行

    - その 1
    
        ```bash
        cargo run
        ```

    - その 2

        ```bash
        cargo build --release
        ./target/release/rust_postgres
        ```
