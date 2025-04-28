BSD 仮想マシン (まだ起動できた姿を見ていない。。。)
===

!!! warning

    CUI ではインストーラまでたどり着けない。。。  
    Virt Manager をインストールしてみたけど、GUI コンソールにも接続できない。。。

    - virt-manager のインストール

        ```bash
        sudo apt install -y virt-manager
        ```


---

以下、やってみたこと。最後までいきつけなかったけど。。。

## FreeBSD

1. インストーラをコンソールブートに固定する

    - iso ファイルのコピーを作成

        ```bash
        sudo mount -o loop /tmp/FreeBSD-14.1-RELEASE-amd64-bootonly.iso /mnt/
        mkdir /tmp/freebsd-iso
        sudo cp -r /mnt/* /tmp/freebsd-iso
        sudo cp -r /mnt/.disk /tmp/freebsd-iso
        ```

    - /tmp/freebsd-iso/boot/loader.conf に下記の設定を追加

        ```text
        echo 'console="comconsole"' | sudo tee /tmp/freebsd-iso/boot/loader.conf
        ```

    - iso ファイルの作成

        ```bash
        sudo apt install genisoimage

        cd /tmp/freebsd-iso
        sudo mkisofs \
            -o /tmp/FreeBSD-14.1-RELEASE-amd64-bootonly-custom.iso \
            -b boot/cdboot \
            -no-emul-boot \
            -r -J -c boot/boot.catalog \
            -boot-load-size 4 \
            -boot-info-table .
        ```

2. `--os-variant` の設定の取得

    ```bash
    osinfo-query os | grep freebsd
    ```

3. ISO イメージの起動

    ```bash
    virt-install \
        --name freebsd14.1 \
        --arch=x86_64 \
        --hvm \
        --ram 4096 \
        --disk size=20 \
        --vcpus 2 \
        --os-variant freebsd13.0 \
        --network bridge=virbr1 \
        --nographics \
        --serial pty -v \
        --cdrom /tmp/FreeBSD-14.1-RELEASE-amd64-bootonly-custom.iso
    ```

## NetBSD

1. `--os-variant` の設定の取得

    ```bash
    osinfo-query os | grep netbsd
    ```

2. ISO イメージの起動

    ```bash
    virt-install \
        --name netbsd10 \
        --arch=x86_64 \
        --hvm \
        --ram 4096 \
        --disk size=20 \
        --vcpus 2 \
        --os-variant netbsd9.0 \
        --network bridge=virbr1 \
        --video=vga \
        --graphics none \
        --console pty,target_type=serial \
        --cdrom /tmp/NetBSD-10.0-amd64.iso
    ```
