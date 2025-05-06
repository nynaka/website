LAN の回線速度調査
===

## iperf3

### インストール

- Linux

    - apt

        ```bash
        sudo apt install iperf3
        ```

    - yam / dnf

        ```bash
        sudo dnf install iperf3
        ```

- macOS

    ```bash
    brew install iperf3
    ```

- Windows

    [iPerf - Download iPerf3 and original iPerf pre-compiled binaries](https://iperf.fr/iperf-download.php) から Windows 版をダウンロードできます。


### 使い方

1. サーバ側でサーバアプリを起動

    ```bash
    iperf3 -s
    ```

2. クライアント側でクライアントアプリを実行

    ```bash
    ipref3 -c サーバの FQDN or IP アドレス
    ```


## Samba のファイル共有速度

1. Samba の共有フォルダを **/mnt/samba** 等にマウントしておきます。
2. dd コマンドで適当なサイズのファイル (ここでは **tmp.bin**) を作成します。

    ```bash
    dd if=/dev/zero of=/mnt/samba/tmp.bin bs=1m count=1000
    ```

    すると、

    ```text
    1000+0 records in
    1000+0 records out
    1048576000 bytes transferred in 11.750280 secs (89238384 bytes/sec)
    ```

    となり、おおよそ 85 MiB 程度だとわかります。
