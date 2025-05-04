[VyOS](https://vyos.io/)
===

ChatGPT によると、

> VyOSは、オープンソースのネットワーク仮想化プラットフォームであり、ルーティング、ファイアウォール、VPNなどの機能を提供します。柔軟性に富み、堅牢なセキュリティを備え、ネットワークの設定と管理を効率化します。VyOSは、小規模なオフィスから大規模なデータセンターまで、さまざまな環境で使用されています。

とのことです。  
業務用ルータやスイッチを扱ったことがある人にとっては、ソフトウェア版のルータ or L3 スイッチといわれた方が、イメージできると思います。  
使い勝手としては、Cisco IOS よりも、20年くらい前の 日立／AlaxalA ルータ？ と思わせるような感じの CLI のような印象です。

## インストール iso イメージファイルの取得

```bash
wget \
    https://github.com/vyos/vyos-nightly-build/releases/download/2025.05.04-0021-rolling/vyos-2025.05.04-0021-rolling-generic-amd64.iso \
    -O /tmp/vyos-2025.05.04-0021-rolling-generic-amd64.iso
```

## VyOS のインストール

1. VyOS にログイン

    ログイン ID / パスワードはどちらも **vyos** です。

2. VyOS のインストール

    ```bash
    install image
    ```

    デフォルト設定で良い場合は、ほとんどの設定項目はデフォルト値のままで動くみたいです。  
    再起動すると ISO イメージではなく、ディスクにインストールされた VyOS が起動します。

    ```
    reboot
    ```


## 設定

VyOS ログインして下記の設定をします。

- 各インタフェースの設定
- DHCP 設定
- NAPT 設定
- SSH ログイン設定

### 設定例

```bash
$ configure
# set interfaces ethernet eth0 address dhcp
# set interfaces ethernet eth1 address 192.168.123.1/24

# set service dhcp-server shared-network-name dhcp_eth1 subnet 192.168.123.0/24 subnet-id 5963
# set service dhcp-server shared-network-name dhcp_eth1 subnet 192.168.123.0/24 range 0 start 192.168.123.101
# set service dhcp-server shared-network-name dhcp_eth1 subnet 192.168.123.0/24 range 0 stop 192.168.123.200
# set service dhcp-server shared-network-name dhcp_eth1 subnet 192.168.123.0/24 option default-router 192.168.123.1
# set service dhcp-server shared-network-name dhcp_eth1 subnet 192.168.123.0/24 option name-server 8.8.8.8

# set nat source rule 1 translation address masquerade
# set nat source rule 1 outbound-interface name eth0
# set nat source rule 1 source address 192.168.123.0/24

# set service ssh
#set system login user vyos authentication public-keys vyos-key key "AAAAC3NzaC1lZ..."
#set system login user vyos authentication public-keys vyos-key type ssh-ed25519

# commit
# save
```

### SSH 公開鍵の設定方法

- キーペアの作成
    - ed25519

        ```bash
        ssh-keygen -t ed25519 -C "comment"
        ```

    - RSA

        ```bash
        ssh-keygen -t rsa -b 4096
        ```

- 公開鍵の登録

    上記コマンドでキーペアを作成すると、公開鍵は $HOME/.ssh/id_ed25519.pub/id_rsa.pub に下記の形式で出力されます。

    ```text title="公開鍵の形式"
    鍵タイプ 公開鍵の値 コメント
    ```

    VyOS への公開鍵の登録は、下記のコマンドで行います。

    ```bash
    set system login user vyos authentication public-keys コメント key "公開鍵の値"
    set system login user vyos authentication public-keys コメント type 鍵タイプ
    ```
