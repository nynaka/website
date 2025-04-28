SysStat
===

## インストール

- Debian Linux

    ```bash
    sudo apt install -y atop sysstat
    ```

    - データ収集間隔を 1 分間隔に変更する

        ```bash
        sudo sed -i 's/^LOGINTERVAL=600.*/LOGINTERVAL=60/' /usr/share/atop/atop.daily
        sudo sed -i -e 's|5-55/10|*/1|' \
                    -e 's|every 10 minutes|every 1 minute|' \
                    -e 's|debian-sa1|debian-sa1 -S XALL|g' /etc/cron.d/sysstat
        sudo sed -i 's/ENABLED=\"false\"/ENABLED=\"true\"/' /etc/default/sysstat
        sudo bash -c "echo 'SA1_OPTIONS=\"-S XALL\"' >> /etc/default/sysstat"
        ```

    この後、`[ ! -d /run/systemd/system ] || exit 0` で `exit 0` に向かってしまうためコメントに修正します。

    ```bash
    #!/bin/sh
    # vim:ts=2:et
    # Debian sa1 helper which is run from cron.d job, not to needlessly
    # fill logs (see Bug#499461).

    set -e

    # Skip in favour of systemd timer
    ### この行 [ ! -d /run/systemd/system ] || exit 0

    # Our configuration file
    DEFAULT=/etc/default/sysstat
    # Default setting, overridden in the above file
    ENABLED=false

    # Read defaults file
    [ ! -r "$DEFAULT" ] || . "$DEFAULT"

    [ "$ENABLED" = "true" ] || exit 0

    exec /usr/lib/sysstat/sa1 "$@"
    ```


## sar コマンド引数

### CPU 使用率

| コマンド       | 出力内容                                  |
| :------------- | :---------------------------------------- |
| sar (引数無し) | 全 CPU の CPU 使用率を出力する            |
| sar -P ALL     | 全 CPU と コアごとの CPU 使用率を出力する |


| 項目    | 説明                                                                             |
| :------ | :------------------------------------------------------------------------------- |
| %user   | ユーザ(アプリケーション)が使用しているCPU使用率                                  |
| %nice   | nice 値が変更されたプロセスがCPUを使用した時間の割合                             |
| %system | カーネルがCPUを使用している時間の割合                                            |
| %iowait | ディスクI/O待ちの時間の割合                                                      |
| %steal  | 仮想環境を使用している環境においてゲストOPSがCPUを割り当てられなかった時間の割合 |
| %idle   | CPUが処理待ちの状態の時間の割合                                                  |


### メモリ使用量

| コマンド | 出力内容               |
| :------- | :--------------------- |
| sar -r   | メモリ使用量を出力する |

| 項目      | 説明                                                                   |
| :-------- | :--------------------------------------------------------------------- |
| kbmemfree | 空きメモリの容量(KB)                                                   |
| kbavail   | 利用可能なメモリの容量(KB)                                             |
| kbmemused | 使用中のメモリの容量(KB)                                               |
| %memused  | メモリの使用率                                                         |
| kbbuffers | バッファの使用量(KB)                                                   |
| kbcached  | キャッシュの使用量(KB)                                                 |
| kbcommit  | システムの動作に必要な事前に確保されているメモリ                       |
| %commit   | 総メモリ容量に対する、システムの動作に必要なメモリの割合               |
| kbactive  | 最近使用されたメモリで、必要のない限り再利用されないメモリの容量(KB)   |
| kbinact   | 最近使用されたメモリで、ほかに用途があれば再利用されるメモリの容量(KB) |
| kbdirty   | ディスクに書き戻されるのを待機しているメモリの容量(KB)                 |

### ネットワーク通信量

| コマンド   | 出力内容                                 |
| :--------- | ---------------------------------------- |
| sar -n DEV | 各ネットワークデバイスの通信量を出力する |

Docker や、その他仮想化を利用していると大量のデータが出力されるので、grep コマンドでデバイスごとに絞り込んだ方が見易いと思います。

| 項目     | 説明                                      |
| :------- | ----------------------------------------- |
| rxpck/s  | 1秒間当たりの受信パケット数               |
| txpck/s  | 1秒間当たりの送信パケット数               |
| rxkB/s   | 1秒間当たりの受信パケットサイズ(KB)       |
| txkB/s   | 1秒間当たりの送信パケットサイズ(KB)       |
| rxcmp/s  | 1秒間当たりの受信圧縮パケット数           |
| txcmp/s  | 1秒間当たりの送信圧縮パケット数           |
| rxmcst/s | 1秒間当たりの受信マルチキャストパケット数 |
| %ifutil  | ネットワークインタフェースの利用率        |


### ディスク I / O

| コマンド | 出力内容                    |
| :------- | --------------------------- |
| sar -b   | ディスク I/O のリクエスト数 |

| 項目    | 説明                                 |
| :------ | ------------------------------------ |
| tps     | 1秒間当たりのI/Oリクエスト数         |
| rtps    | 1秒間当たりの読み込みI/Oリクエスト数 |
| wtps    | 1秒間当たりの書き込みI/Oリクエスト数 |
| bread/s | 1秒間当たりの読み込みブロック数      |
| bwrtn/s | 1秒間当たりの書き込みブロック数      |

### スワップ使用量

| コマンド | 出力内容             |
| :------- | -------------------- |
| sar -S   | スワップメモリ使用量 |

| 項目      | 説明                                 |
| :-------- | :----------------------------------- |
| kbswpfree | スワップ領域の空き容量(KB)           |
| kbswpused | スワップ領域の使用量(KB)             |
| %swpused  | スワップ領域の使用率                 |
| kbswpcad  | スワップ領域のキャッシュの使用量(KB) |
| %swpcad   | スワップ領域のキャッシュの使用率     |


## 参考サイト

- [Amazon Linux、RHEL、CentOS、または Ubuntu を実行している自分の EC2 インスタンスに対して、ATOP および SAR モニタリングツールを設定する方法を教えてください。](https://aws.amazon.com/jp/premiumsupport/knowledge-center/ec2-linux-configure-monitoring-tools/)
