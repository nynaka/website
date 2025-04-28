[QEMU](https://www.qemu.org/) 単体の仮想化
===

QEMU では KVM と連携しなくても単体で完全仮想化型の環境を提供できます。  
ただ、私の実行環境では、VMware Workstation や VirtualBox と比べて、ものすごく動作が遅いので、KVM と組み合わせないと実用的ではないのかもしれない、と思い始めています。

## QEMU のインストール

- Debian / Ubuntu

    ```bash
    sudo apt install -y qemu-system
    ```

## 仮想マシンの GUI インストール

仮想マシンの GUI インストールは下記の流れで実行します。  
自分の環境が VMware Workstation Player 上の Linux で実行しているせいか、ものすごく遅いです。

1. 仮想ディスクの作成

    ```bash
    qemu-img create \
        -f qcow2 \
        debian.qcow2 \
        32G
    ```

2. CD/DVD イメージからのインストール

    ```bash
    qemu-system-x86_64 \
        -accel tcg \
        -boot menu=on \
        -cdrom debian-12.10.0-amd64-netinst.iso \
        -m 4G \
        debian.qcow2
    ```

    - このコマンドを実行すると VirtualBox や VMware のように、仮想マシンの画面が起動します。
    - 普段は目にしないインストールメニューが表示され、手動選択しないと次に進まない状態になることがあるようですが、気にせずインストールを進めます。ただ、インストールファイルの展開がすごい遅い。

3. インストールした OS の起動

    ```bash
    qemu-system-x86_64 \
        -accel tcg \
        -m 4G \
        -nic \
        user,hostfwd=tcp::60022-:22 \
        debian.qcow2
    ```


## 参考サイト

- [QEMU - ArchWiki](https://wiki.archlinux.jp/index.php/QEMU)
