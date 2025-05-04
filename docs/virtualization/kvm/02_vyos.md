[VyOS](https://vyos.io/)
===

ChatGPT によると、

> VyOSは、オープンソースのネットワーク仮想化プラットフォームであり、ルーティング、ファイアウォール、VPNなどの機能を提供します。柔軟性に富み、堅牢なセキュリティを備え、ネットワークの設定と管理を効率化します。VyOSは、小規模なオフィスから大規模なデータセンターまで、さまざまな環境で使用されています。

とのことです。  
業務用ルータやスイッチを扱ったことがある人にとっては、ソフトウェア版のルータ or L3 スイッチといわれた方が、イメージできると思います。  
使い勝手としては、Cisco IOS よりも、20年くらい前の 日立／AlaxalA ルータ？ と思わせるような感じの CLI のような印象です。

## インストール

1. インストール iso イメージファイルの取得

    ```bash
    wget \
        https://github.com/vyos/vyos-nightly-build/releases/download/2025.03.18-0018-rolling/vyos-2025.03.18-0018-rolling-generic-amd64.iso \
        -O /tmp/vyos-2025.03.18-0018-rolling-generic-amd64.iso
    ```

2. VyOS の起動

    ```bash
    virt-install \
        --name vyos \
        --arch=x86_64 \
        --hvm \
        --ram 1024 \
        --disk size=10 \
        --vcpus 1 \
        --os-variant generic \
        --network bridge=virbr0,model=virtio \
        --network bridge=virbr1,model=virtio \
        --graphics none \
        --console pty,target_type=serial \
        --cdrom=/tmp/vyos-2025.03.18-0018-rolling-generic-amd64.iso
    ```

3. VyOS に接続

    項番 2 の起動コマンド実行後、しばらくするとログインプロンプトが表示されます。  
    うっかり `Ctrl+]` で抜けてしまった場合は下記コマンドでコンソールに接続してください。

    ```bash
    virsh console vyos
    ```

    デフォルトのログイン ID / パスワードはどちらも **vyos** です。


4. VyOS のインストール

    ```bash
    install image
    ```

    デフォルト設定で良い場合は、ほとんどの設定項目はデフォルト値のままで動くみたいです。  
    再起動すると ISO イメージではなく、ディスクにインストールされた VyOS が起動します。

    ```
    reboot
    ```

    で再起動してくれるはずなのですが、再起動してくれない場合は下記コマンドを実行して起動します。

    ```bash
    virsh start vyos
    ```

    vyos は Gateway という性格上、下記コマンドを実行して自動起動にしておくと、OS 起動ごとに再起動する手間が省けます。

    ```bash
    virsh autostart vyos
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

# commit
# save
```
