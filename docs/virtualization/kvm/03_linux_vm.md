Linux 仮想マシン
===

## Debian Linux 12

1. OS インストール ISO ファイルの取得

    ```bash
    wget https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-12.10.0-amd64-netinst.iso \
        -O /tmp/debian-12.10.0-amd64-netinst.iso
    ```

2. `--os-variant` の設定の取得

    ```bash
    osinfo-query os | grep debian
    ```

3. ISO イメージの起動

    ```bash
    virt-install \
        --name debian12 \
        --hvm \
        --ram 2048 \
        --disk size=20 \
        --vcpus 2 \
        --os-variant debian12 \
        --graphics none \
        --network bridge=virbr1 \
        --console pty,target_type=serial \
        --extra-args 'console=ttyS0,115200n8 serial' \
        --location /tmp/debian-12.10.0-amd64-netinst.iso,kernel=install.amd/vmlinuz,initrd=install.amd/initrd.gz
    ```

    GUI 無しでインストールする場合は、インストーラのカーネルに console 関連の引数を追加する都合で、ISO イメージのファイルパスの他に、ISO イメージ内のインストーラのカーネルパスと initrd のパスを指定する必要があります。

    - ネットワークインストールする場合

        ```bash
        sudo virt-install \
            --name debian12 \
            --hvm \
            --ram 2048 \
            --disk size=20 \
            --vcpus 2 \
            --os-variant debian12 \
            --graphics none \
            --network bridge=virbr0 \
            --console pty,target_type=serial \
            --extra-args 'console=ttyS0,115200n8 serial' \
            --location 'http://ftp.debian.org/debian/dists/stable/main/installer-amd64/'
        ```


## Fedora Server 41

1. `--os-variant` の設定の取得

    ```bash
    osinfo-query os | grep fedora
    ```

2. ISO イメージの起動

    ```bash
    virt-install \
        --name fedora41 \
        --arch=x86_64 \
        --hvm \
        --ram 4096 \
        --disk size=20 \
        --vcpus 2 \
        --os-variant fedora40 \
        --network bridge=virbr1 \
        --graphics none \
        --console pty,target_type=serial \
        --extra-args 'console=ttyS0,115200n8 serial' \
        --location /tmp/Fedora-Server-netinst-x86_64-41-1.4.iso
    ```

    Fedora server では、`--location` に ISO イメージ内のインストーラのカーネルパスと initrd のパスを指定する必要はないのですが、Fedora のテキストベースのインストーラには戸惑うと思います。


## Ubuntu Server 24.04

1. `--os-variant` の設定の取得

    ```bash
    osinfo-query os | grep ubuntu
    ```

2. ISO イメージの起動

    ```bash
    virt-install \
        --name ubuntu2404 \
        --arch=x86_64 \
        --hvm \
        --ram 4096 \
        --disk size=20 \
        --vcpus 2 \
        --os-variant ubuntu24.04 \
        --network bridge=virbr1 \
        --graphics none \
        --console pty,target_type=serial \
        --extra-args 'console=ttyS0,115200n8 serial' \
        --location /tmp/ubuntu-24.04.2-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd
    ```

    GUI 無しでインストールする場合は、インストーラのカーネルに console 関連の引数を追加する都合で、ISO イメージのファイルパスの他に、ISO イメージ内のインストーラのカーネルパスと initrd のパスを指定する必要があります。


## コマンドオプション

- --name : 仮想マシンの名前
- --arch : CPU のアーキテクチャ
- --hvm
- --vcpus : 仮想 CPU の数
- --ram : メモリ量（MB）
- --disk : ディスクサイズ（GB）
- --os-variant
- --network : network=default: ネットワークの設定
- --graphics
- --console : pty,target_type=serial
- --extra-args : 'console=ttyS0,115200n8'
- --location : ISO ファイルと、インストーラが格納されているパスに相当する情報
