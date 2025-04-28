KVM コマンド操作
===


### VM の起動

* 停止中の VM の確認

    ```bash
    sudo virsh list --all

    Id   Name         State
    -----------------------------
    -    ubuntu2204   shut off
    ```

* VM の起動

    ```bash
    sudo virsh start ubuntu2204
    ```

* VM のコンソールに接続

    ```bash
    sudo virsh console ubuntu2204
    ```