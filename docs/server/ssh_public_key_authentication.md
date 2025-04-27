SSH 公開鍵認証設定
===

## 設定

- 鍵作成

    - RSA 鍵

        ```bash
        ssh-keygen -t rsa -b 4096
        ```

        上記コマンドを実行すると `$HOME/.ssh` 配下に、秘密鍵 (id_rsa) と公開鍵 (id_rsa.pub) が生成されます。

    - ED 25519 鍵

        ```bash
        ssh-keygen -t ed25519
        ```

        上記コマンドを実行すると `$HOME/.ssh` 配下に、秘密鍵 (id_ed25519) と公開鍵 (id_ed25519.pub) が生成されます。

- 公開鍵の扱い

    $HOME/.ssh/authorized_keys に登録します。  
    このファイル名は /etc/ssh/sshd_config で変更できます。

    ```bash
    # RSA 鍵
    cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys
    # ED 25519 鍵
    cat $HOME/.ssh/id_ed25519.pub >> $HOME/.ssh/authorized_keys
    ```

- 秘密鍵の扱い

    安全な方法で SSH クライアントにコピーします。秘密鍵の安全なコピー手段を検討するより、SSH クライアントで鍵を生成して公開鍵を SSH サーバにコピーした方が安全性は高いと思います。

- SSH クライアントコンフィグ

    下記のように設定ファイルを用意すると、`ssh ssh_server` のように `Host` で設定した名前指定で SSH ログインできるようになります。

    ```text title="$HOME/.ssh/config"
    Host ssh_server
        HostName localhost
        User debian
        Port 22
        IdentityFile /home/debian/.ssh/id_rsa
    ```


## 稀によくある問題

- SSH 接続が一定時間で切断される

    SSH では一定時間、クライアントから応答がないと自動的に切断する機能があり、デフォルト値 (ClientAliveInterval=0) の場合、応答確認は行わずに切断する、という設定なのですが、たいていはかなりの時間、接続が維持されます。  
    ただ、よくわからないタイミングで切断されることを回避したい場合は、下記のように ClientAliveInterval と ClientAliveCountMax を設定すると `ClientAliveInterval - ClientAliveCountMax` 秒間、無応答でも接続が維持されるようになります。

    ```text title="/etc/ssh/sshd_config"
    # ClientAliveInterval 0
    # ClientAliveCountMax 3
    ↓
    ClientAliveInterval 60
    ClientAliveCountMax 20
    ```
