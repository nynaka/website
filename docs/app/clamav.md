ClamAV
===

## Debian / Ubuntu

- インストール

    ```bash
    sudo apt install -y clamav clamav-daemon
    ```

- clamav-daemon の起動設定

    ```bash
    sudo systemctl start clamav-daemon.service
    sudo systemctl enable clamav-daemon.service
    ```

- 自動ウイルス定義更新サービスの起動設定

    ```bash
    sudo systemctl start clamav-freshclam.service 
    sudo systemctl enable clamav-freshclam.service
    ```

- 手動スキャン

    コマンドが 2 つインストールされる。

    - clamscan

        - clamav-daemon 不要
        - コマンドオプションが豊富
        - 処理に時間がかかる
        - コマンド実行例

            ```bash
            sudo clamscan /path/to/scan/target
            ```

    - clamdscan

        - clamav-daemon 必要
        - 処理が高速 (マルチスレッドだから？)
        - コマンド実行例

            ```bash
            sudo clamdscan /path/to/scan/target
            ```

        - 定期実行を想定する場合

            ```bash
            sudo mkdir -p /var/clanmav/{virus,history}
            sudo clamdscan \
                --recursive \
                --infected \
                --multiscan \
                --fdpass \
                --move="/var/clanmav/virus" \
                --log="/var/clanmav/history/$(date +%Y%m%d-%H%M%S).log" /home
            ```

- GUI ツール

    ```bash
    sudo apt install clamtk
    ```
