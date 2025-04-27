MacOS Tips
===

## ISO ファイルを USB メモリに書き込む

Linux のインストール ISO ファイルを前提とした手順ですが、RaspberryPi OS イメージのインストールも同じ手順で実行できます。

1. USB メモリのデバイスファイルの確認

    ```bash
    diskutil list
    ```

2. USB メモリのアンマウント

    USB メモリのデバイスファイルパスが `/dev/disk4` とする。

    ```bash
    diskutil unmountDisk /dev/disk4
    ```

3. ISO ファイルを USB メモリに書き込む

    カレントディレクトリに保存している `xubuntu-22.04.5-desktop-amd64.iso` を USB メモリに書き込みます。

    ```bash
    sudo dd \
        if=./xubuntu-22.04.5-desktop-amd64.iso \
        of=/dev/rdisk4 \
        bs=1m \
        status=progress
    ```

4. USB メモリの取り外し

    ```bash
    diskutil eject /dev/disk4
    ```

