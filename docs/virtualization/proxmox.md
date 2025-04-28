Proxmox
===

## 設定

* Web インタフェースの URL: `https://IPアドレスまたはFQDN:8006/`


### サブスクリプションなしの apt リポジトリ設定

1. /etc/apt/sources.list.d/pve-enterprise.list の編集

    ```diff
    - deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
    + #deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
    + deb [arch=amd64] http://download.proxmox.com/debian/pve bullseye pve-no-subscription
    ```

2. /etc/apt/sources.list.d/ceph.list

    ```diff
    - deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise
    + #deb https://enterprise.proxmox.com/debian/ceph-quincy bookworm enterprise
    ```

3. 公開鍵のインストール

    ```bash
    wget https://enterprise.proxmox.com/debian/proxmox-release-bullseye.gpg \
        -O /etc/apt/trusted.gpg.d/proxmox-release-bullseye.gpg 
    ```


### ネスト仮想化の有効化

ハイパーバイザ上の仮想マシンに Intel-VT や AMD-V を見せることを `ネスト仮想化` と呼ぶらしい。

* modprobe の設定

    * Intel-VT

        ```bash
        echo "options kvm-intel nested=Y" | sudo tee /etc/modprobe.d/kvm-intel.conf
        ```

    * AMD-V

        ```bash
        echo "options kvm-amd nested=1" | sudo tee /etc/modprobe.d/kvm-amd.conf
        ```

* Kernel module を再読み込み

    * Intel-VT

        ```bash
        sudo modprobe -r kvm_intel
        sudo modprobe kvm_intel
        ```

    * AMD-V

        ```bash
        sudo modprobe -r kvm_amd
        sudo modprobe kvm_amd
        ```

* module の設定反映確認

    * Intel-VT
    
        ```bash
        cat /sys/module/kvm_intel/parameters/nested
        ```

    * AMD-V

        ```bash
        cat /sys/module/kvm_amd/parameters/nested
        ```

* 仮想マシンの設定

    CPU の `Type` を `x86-64-v2-AES` 等 ⇒ `host` に変更します。

* 仮想マシンから Intel-VT／AMD-V を利用できることの確認

    ```bash
    egrep -c '(vmx|svm)' /proc/cpuinfo
    ```

    上記コマンドを実行すると、仮想マシンに設定した CPU のコアの総数が表示されると思います。


## 一般的な Debian Linux として利用する

### 基本設定

* ツールのインストール

    ```bash
    apt install -y sudo locales man manpages-ja ssh \
        curl wget zip unzip bzip2 p7zip-full csh zsh expect \
        vim emacs \
        git subversion jq \
        cryptsetup
    ```

* 一般ユーザ登録

    ```bash
    useradd -u 1000 -g 100 -G sudo -s /bin/bash -d /home/debian -m debian
    passwd debian
    ```

* Samba

    ```bash
    sudo apt install samba
    ```

    ```bash
    sudo smbpasswd -a debian
    ```

    ```bash
    sudo systemctl start smbd nmbd
    sudo systemctl enable smbd nmbd
    ```


* Wifi インタフェース

    * ツールのインストール

        ```bash
        sudo apt install wireless-tools wpasupplicant
        ```

    * インタフェースの動作確認ついでの ESSID スキャン

        無線 LAN インタフェースのインタフェース名は `wlp2s0` としています。

        ```bash
        iwlist wlp2s0 scanning
        ```

    * wpa_supplicant.conf の作成

        ```bash
        wpa_passphrase ESSID password
        ```

        の結果の先頭に、

        ```text
        ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
        update_config=1
        country=JP
        ```
        
        を追加した内容で、`/etc/wpa_supplicant/wpa_supplicant.conf` を作成する。具体的には、下記のような感じ。

        ```conf
        ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
        update_config=1
        country=JP

        network={
                ssid="ESSID"
                #psk="password"  平文パスワードは消しましょう
                psk=907998845ecfdc66f243ac57e6a5aab06c864049caf01a671a179358da6808e1
        }
        ```

    * /etc/network/interfaces の変更

        ```conf
        #iface wlp2s0 inet manual
        auto wlp2s0
        iface wlp2s0 inet dhcp
            wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
        ```


* NAPT の設定

    無線 LAN をインターネット方法、有線 LAN を内部ネットワークとする NAPT 設定とする。

    * net.ipv4.ip_forward の設定

        ```bash
        echo "net.ipv4.ip_forward=1" | sudo tee >> /etc/sysctl.conf
        ```

    * NAPT 設定

        * ファイアウォールの設定

        ```bash
        #!/bin/sh

        # 既存設定の初期化
        iptables -F
        iptables -F -t nat
        iptables -X

        # Deafult Rule
        iptables -P INPUT   ACCEPT
        iptables -P OUTPUT  ACCEPT
        iptables -P FORWARD ACCEPT

        iptables -t nat -A POSTROUTING -o wlp2s0 -j MASQUERADE
        ```

        * ファイアウォール設定の永続化

            ```bash
            sudo apt install -y iptables-persistent
            sudo /etc/init.d/netfilter-persistent save
            ```

* DHCP サーバ設定

    * ISC DHCP サーバのインストール

        ```bash
        sudo apt install isc-dhcp-server
        ```

    * /etc/default/isc-dhcp-server

        ```diff
        @@ -14,5 +14,5 @@
        
         # On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
         #      Separate multiple interfaces with spaces, e.g. "eth0 eth1".
        -#INTERFACESv4=""
        +INTERFACESv4="vmbr0"
         #INTERFACESv6=""
        ```

    * /etc/dhcp/dhcpd.conf

        ```conf
        # These options are used to add static routes
        option rfc3442-classless-static-routes code 121 = array of integer 8;
        option ms-classless-static-routes code 249 = array of integer 8;

        subnet 192.168.1.0 netmask 255.255.255.0 {
            range 192.168.1.101 192.168.1.200;
            option routers 192.168.1.1;
            option domain-name-servers 8.8.8.8,8.8.4.4;
            # Route                             to    172.16.10.0/24 through 192.168.1.1;
            #                                       ┌──────┘  ┌───┘
            #                                       |  |   |  |       |
            option rfc3442-classless-static-routes 24, 172,16,10, 192,168,1,1;
        }
        ```

    * 起動設定

        ```bash
        sudo systemctl enable isc-dhcp-server
        sudo systemctl start isc-dhcp-server
        ```
