Samba
===

## インストール

- Debian Linux

    ```bash
    sudo apt install -y samba
    ```

- デーモン起動制御

    | action       | command                      |
    | :----------- | :--------------------------- |
    | 起動         | sudo systemctl start samba   |
    | 停止         | sudo systemctl stop samba    |
    | 再起動       | sudo systemctl restart samba |
    | 自動起動     | sudo systemctl enable samba  |
    | 起動状態確認 | sudo systemctl status samba  |

## 基本設定

- /etc/samba/smb.conf

    ```ini
    [global]
    # Windowsと通信するときに使用する文字コード
    dos charset = CP932
    # Unixと通信するときに使用する文字コード
    unix charset = utf-8
    # macOSと通信するときに有効にする...らしい
    unix extensions = yes

    # ワークグループ名
    workgroup = WORKGROUP
    server string = Samba Server %v
    netbios name = %h
    security = user
    map to guest = bad user
    dns proxy = no

    # samba パスワード変更時、Linux パスワードも変更する
    unix password sync = yes
    passwd program = /usr/bin/passwd %u
    passwd chat = *new*password* %n\n *new*password* %n\n *changed*

    ### Debug ###
    # 出力ログレベル。数値が大きいほど詳細なログを出力する
    log level = 1

    # ログファイル出力先
    log file = /var/log/samba/log.%m

    ### 認証 ###
    # ゲストユーザの無効化。
    # ゲストユーザを許可する場合、「never」->「bad user」に変更する
    map to guest = never

    #==========================================================

    [homes]
    comment = Home Directiry
    #マイネットワークに表示させるか否か
    browseable = No
    #書き込み可能かどうか
    writable = Yes
    valid user = %U
    # オフラインキャッシュは無効
    csc policy = disable

    create mask = 0744
    directory mask = 0755
    ```

- Samba アカウント登録

    ```bash
    sudo smbpasswd -a debian
    ```


## Windows エクスプローラの `以前のバージョン` を利用する

LVM や ZFS、BTRFS を利用すると、割と簡単に `以前のバージョン` 機能を利用できます。  
ここでは BTRFS を使用します。

- BTRFS 管理ツールのインストール

    ```bash
    sudo apt install -y btrfs-progs
    ```

- BTRFS ファイルシステムの準備

    `/dev/sdb` 全体を BTRFS ファイルシステムとして利用することを前提にします。

    - パーティション作成

        ```bash
        sudo apt install parted
        sudo parted /dev/sdb --script mklabel gpt mkpart primary btrfs 0% 100%
        ```

    - BTRFS ファイルシステム作成

        ```bash
        sudo mkfs.btrfs /dev/sdb1
        ```

    - ファイルシステムのマウント

        - マウントポイント作成

            ```bash
            sudo mkdir /sdb
            ```

        - /etc/fstab の設定

            ```bash
            echo "/dev/sdb1    /sdb    btrfs    noatime,nodiratime,compress=lzo    0 0" | sudo tee -a /etc/fstab
            ```

        - mount

            ```bash
            sudo systemctl daemon-reload
            sudo mount -a
            ```

- BTRFS サブボリューム作成

    ```bash
    sudo btrfs subvolume create /sdb/samba
    ```

- smb.conf の変更

    ```diff
    --- /tmp/smb.conf       2025-01-25 13:31:53.436066099 +0900
    +++ /etc/samba/smb.conf 2025-01-25 14:12:51.012156450 +0900
    @@ -46,3 +46,28 @@
     create mask = 0744
     directory mask = 0755
    
    +[samba]
    +path = /sdb/samba
    +comment = Samba Directory
    +#マイネットワークに表示させるか否か
    +browsable =yes
    +#ゲストユーザのログインが可能かどうか
    +guest ok = no
    +#書き込み可能かどうか
    +writable = yes
    +#読込みのみとするか
    +read only = no
    +
    +# オフラインキャッシュは無効
    +csc policy = disable
    +
    +create mask = 0744
    +directory mask = 0755
    +
    +vfs objects = shadow_copy2
    +shadow:snapdir = /sdb/snapshots/
    +shadow:basedir = /sdb/samba/
    +shadow:sort = desc
    +shadow:format = %Y-%m-%d_%H-%M-%S
    +shadow:localtime = yes
    ```

    - `vfs objects = shadow_copy2` 以下の行が重要です。
    - smb.conf 変更後、Sambaサーバを再起動します。

- 動作確認

    - スナップショットの作成

        ```bash
        sudo mkdir -p /sdb/snapshots/
        sudo btrfs subvolume snapshot /sdb/samba /sdb/snapshots/$(date +"%Y-%m-%d_%H-%M-%S")
        ```

    - smbclient によるスナップショットの確認

        - Sambaサーバにログイン

            ```bash
            smbclient //localhost/samba -U debian
            ```

            - `debian` はユーザ名で、コマンド実行後パスワードの入力を求められます。

        - smbclient コマンドの実行

            ```bash
            allinfo ファイルやディレクトリ名
            ```

    Windows エクスプローラーの `以前のバージョン` に表示されるようになるタイミングが、いまいちわかりませんが、スナップショット後、ファイルやフォルダを変更すると表示されるようになる気がします。

### 自動スナップショット

- スナップショット管理シェル (snapshot_btrfs_dir.sh)

    ```bash
    BASE_DIR="/sdb/samba"
    SNAPSHOT_DIR="/sdb/snapshots"
    NUM_OF_HISTORIES=10

    # スナップショットの作成
    sudo btrfs subvolume snapshot ${BASE_DIR} ${SNAPSHOT_DIR}/$(date +"%Y-%m-%d_%H-%M-%S")

    # 古いスナップショットの削除
    SNAPSHOT=($(ls ${SNAPSHOT_DIR} | sort -r))
    if [ ! -z "${SNAPSHOT}" ]; then
        for ((i=${NUM_OF_HISTORIES}; i<${#SNAPSHOT[@]}; i++)); do
            sudo btrfs subvolume delete ${SNAPSHOT_DIR}/${SNAPSHOT[$i]}
        done
    else
        echo "subdirs is empty"
    fi
    ```

- cron の設定

    ```bash
    echo "* * * * * root  bash /root/snapshot_btrfs_dir.sh" | sudo tee -a /etc/crontab
    ```

- cron の再起動

    ```bash
    sudo systemctl restart cron
    ```

    一応、cron の再起動していますが、/etc/crontab を編集した時点で有効になっているような動きをしています。
