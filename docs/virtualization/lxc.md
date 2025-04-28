LXC (Linux Containers)
===

## インストール

- Debian / Ubuntu

    ```bash
    sudo apt install -y lxc
    ```

## 用語の説明

- 特権コンテナ

    権コンテナとは、root 権限で作成し、root 権限で実行するコンテナのことです。


## LXC コンテナの OS テンプレート

LXC の OS テンプレートは [https://images.linuxcontainers.org/](https://images.linuxcontainers.org/) で確認することができます。  
LXC コンテナ作成コマンドで、

```bash
sudo lxc-create --name mycontainer --template download
```

で実行すると、利用可能なコンテナ候補を確認することができます。


## LXC の主な管理コマンド

| 機能                   | コマンド    |
| :--------------------- | :---------- |
| コンテナ作成           | lxc-create  |
| コンテナ起動           | lxc-start   |
| コンテナ停止           | lxc-stop    |
| コンテナ削除           | lxc-destroy |
| コンテナのシェルに接続 | lxc-attach  |

### コンテナの作成

- Debian Linux 12 コンテナの作成

    ```bash
    sudo lxc-create --name debian12 --template download -- \
        --dist debian --release bookworm --arch amd64
    ```

- Fedora Linux 41 コンテナの作成

    ```bash
    sudo lxc-create --name debian12 --template download -- \
        --dist fedora --release 41 --arch amd64
    ```

### コンテナの起動

- コンテナの起動

    ```bash
    sudo lxc-start --name debian12
    ```

- 起動確認 (コンテナ個別)

    ```bash
    sudo lxc-info --name debian12
    ```

- 起動確認 (起動中のコンテナ全部)

    ```bash
    sudo lxc-ls --fancy
    ```

- コンテナの自動起動設定

    LXC コンテナごとの設定ファイルの `lxc.start.auto` に 1 を設定すると、OS のブート時に当該コンテナを自動起動できます。

    ```bash
    echo "lxc.start.auto = 1" | sudo tee -a /var/lib/lxc/debian12/config
    ```

### コンテナ内のシェルに接続

```bash
sudo lxc-attach --name debian12
```

### コンテナ停止

```bash
sudo lxc-stop --name debian12
```

### コンテナの削除

```bash
sudo lxc-destroy --name debian12
```

---

## 参考サイト

- [Linux Containers - LXC - はじめに](https://linuxcontainers.org/ja/lxc/getting-started/)