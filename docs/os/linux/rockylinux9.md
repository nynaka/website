Rocky Linux9
===

!!!note
    RockyLinux9 を最小構成でインストールすることを前提としています。

## 設定

### SELinux 無効化

### SSH

- インストール

    ```bash
    sudo dnf install openssh-server
    ```

- 起動設定

    ```bash
    sudo systemctl enable sshd
    sudo systemctl start sshd
    ```

- ed25519 キーペア作成

    ```bash
    ssh-keygen -t ed25519 -C "rocky@rocky9.local"
    ```

- 秘密鍵から公開鍵を作成する

    ```bash
    ssh-keygen -y -f [秘密鍵のファイルパス] > [作成する公開鍵のファイルパス]
    ```

- 公開鍵の登録

    ```bash
    cat $HOME/.ssh/id_ed25519.pub >> $HOME/.ssh/authorized_keys
    ```

- SSH クライアントの設定

    **秘密鍵 ($HOME/.ssh/id_ed25519)** を SSH クライアントにコピーします。

    - Windows の場合

        C:\Users\ユーザ\\.ssh\ に秘密鍵をコピーします。

        ```text title="C:\Users\ユーザ\.ssh\config"
        Host rocky9.local
            HostName rocky9.local
            IdentityFile C:\Users\ユーザ\.ssh\id_ed25519_rocky9
            User rocky
        ```

### mDNS

- インストール

    ```bash
    sudo dnf install avahi
    ```

- 起動設定

    ```bash
    sudo systemctl enable avahi-daemon
    sudo systemctl start avahi-daemon
    ```

- ファイアウォールの設定

    ```bash
    # 恒久的な設定として追加
    sudo firewall-cmd --permanent --add-service=mdns

    # 設定を反映
    sudo firewall-cmd --reload
    ```


### VMware Workstation Pro 共有フォルダ

- open-vm-tools のインストール

    ```bash
    # パッケージの更新（推奨）
    sudo dnf update -y

    # open-vm-tools と関連ツールのインストール
    sudo dnf install open-vm-tools open-vm-tools-desktop -y

    # サービスが有効か確認し、起動
    sudo systemctl enable --now vmtoolsd
    ```

- VMware の共有フォルダが見えているか確認

    ```bash
    vmware-hgfsclient
    ```

    ここでは **_share** が表示されたものとします。

- マウントポイント作成

    ```bash
    sudo mkdir -p /mnt/hgfs
    ```

- 手動マウント

    ```bash
    sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other -o auto_unmount
    ```

- 自動マウント

    - /etc/fstab に追記

        ```bash
        .host:/    /mnt/hgfs    fuse.vmhgfs-fuse    allow_other,defaults    0    0
        ```

    - マウント

        ```bash
        sudo systemctl daemon-reload
        mount -a
        ```

        /mnt/hgfs/_share/ から共有フォルダに接続できます。


## PostgreSQL

[How to Install PostgreSQL 18 in Rocky Linux | Atlantic.Net](https://www.atlantic.net/dedicated-server-hosting/how-to-install-postgresql-18-in-rocky-linux/)
