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



