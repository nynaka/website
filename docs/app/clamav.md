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
    sudo apt install -y clamtk
    ```


## リアルタイムスキャン

### /etc/clamav/clamd.conf の編集

- 差分（ここでは、`patch.txt` という名前で保存するものとする）

    ```diff
    --- clamd.conf.orig     2022-03-20 23:02:12.795136761 +0000
    +++ clamd.conf  2022-03-20 23:08:03.900661995 +0000
    @@ -3,11 +3,11 @@
     #Please read /usr/share/doc/clamav-daemon/README.Debian.gz for details
     LocalSocket /var/run/clamav/clamd.ctl
     FixStaleSocket true
    -LocalSocketGroup clamav
    +LocalSocketGroup root
     LocalSocketMode 666
     # TemporaryDirectory is not set to its default /tmp here to make overriding
     # the default with environment variables TMPDIR/TMP/TEMP possible
    -User clamav
    +User root
     ScanMail true
     ScanArchive true
     ArchiveBlockEncrypted false
    @@ -64,8 +64,8 @@ ForceToDisk false
     DisableCertCheck false
     DisableCache false
     MaxScanTime 120000
    -MaxScanSize 100M
    -MaxFileSize 25M
    +MaxScanSize 0
    +MaxFileSize 0
     MaxRecursion 16
     MaxFiles 10000
     MaxPartitions 50
    @@ -85,3 +85,10 @@ Bytecode true
     BytecodeSecurity TrustSigned
     BytecodeTimeout 60000
     OnAccessMaxFileSize 5M
    +# Configuration for on-access scan
    +TCPAddr 127.0.0.1
    +TCPSocket 3310
    +OnAccessIncludePath /home
    +OnAccessIncludePath /var/www/html
    +OnAccessExtraScanning yes
    +OnAccessExcludeRootUID yes
    ```

- 差分反映

    ```bash
    cd /etc/clamav/
    sudo patch -p0 < patch.txt
    ```

### 起動スクリプト作成

- 起動スクリプトの内容

    ```ini title="/etc/systemd/system/clamonacc.service"

    [Unit]
    Description=ClamAV On Access Scanner
    Requires=clamav-daemon.service
    After=clamav-daemon.service syslog.target network.target
    
    [Service]
    Type=simple
    User=root
    ExecStartPre=/bin/bash -c "while [ ! -S /var/run/clamav/clamd.ctl ]; do sleep 1; done"
    ExecStart=/usr/sbin/clamonacc -F --config-file=/etc/clamav/clamd.conf --log=/var/log/clamav/onacc.log --move=/root/quarantine
    
    [Install]
    WantedBy=multi-user.target
    ```

- ウイルス検知したファイルの移動先ディレクトリ作成

    ```bash
    sudo mkdir /root/quarantine
    ```

- On-Access Scan の自動起動設定

    ```bash
    sudo systemctl enable clamonacc
    sudo systemctl start clamonacc
    ```

- 動作確認

    - ウィルス検出テストファイルのダウンロード

        ```bash
        cd $HOME
        wget http://www.eicar.org/download/eicar.com
        wget https://secure.eicar.org/eicar.com.txt
        wget https://secure.eicar.org/eicar_com.zip
        ```        

    - /var/log/clamav/onacc.log

        ```text
        ClamInotif: watching '/home' (and all sub-directories)
        ClamInotif: watching '/var/www/html' (and all sub-directories)
        /home/hoge/eicar.com: Win.Test.EICAR_HDB-1 FOUND
        /home/hoge/eicar.com: moved to '/root/quarantine/eicar.com'
        ```
