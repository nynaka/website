FreeBSD 14
===

## 初期設定

* 一般ユーザ (ここでは `freebsd`) で root に昇格できるようにする

    `wheel` グループに所属させるようにすると su コマンドで昇格できるようになります。

    ```csh
    pw user mod freebsd -G wheel
    ```

* SSH ログインでパスワード認証を許可する

    * /etc/ssh/sshd_config の編集

        ```text
        #PasswordAuthentication no
        ↓
        PasswordAuthentication yes
        ```

    * sshd の再起動

        ```csh
        /etc/rc.d/sshd restart
        ```

* sudo コマンド

    * インストール

        ```csh
        pkg install -y sudo
        ```

    * /usr/local/etc/sudoers の編集

        ```csh
        # %wheel ALL=(ALL:ALL) ALL
        ↓
        %wheel ALL=(ALL:ALL) ALL
        ```

    * sudo コマンドで root 昇格できるようにする

        ```csh
        pw user mod freebsd -G wheel
        ```

    `wheel` グループに所属させると sudo コマンドで昇格できるようになります。

* mDNS で名前解決する

    * インストール

        ```csh
        pkg install -y avahi
        ```

        およそ使わないパッケージもインストールされるので、最小限のインストールをする場合はソースでインストールした方が良いようです。

    * /etc/rc.conf に追加

        ```text:
        avahi_daemon_enable="YES"
        dbus_enable="YES"
        ```

    * daemon の起動

        ```csh
        service dbus start
        service avahi-daemon start
        ```

    * FreeBSD で mDNS で解決する名前を利用する

        * nss_mdns をインストールする

            ```csh
            pkg install -y nss_mdns
            ```

        * /etc/nsswitch.conf の編集

            ```csh
            hosts: files dns
            ↓
            hosts: files mdns dns
            ```

* root ユーザのシェルを tcsh に変更する

    ```csh
    pw user mod root -s /bin/tcsh
    ```

* 時刻調整

    ```csh
    sudo ntpdate ntp.nict.jp
    ```

* FreeBSD 設定ツール

    ```
    sudo bsdconfig
    ```

    FreeBSD 10 より古い時代の `sysinstall` コマンドの代替えコマンドです。


## アプリのインストール

### pkg コマンドでインストールしたアプリのアップグレード

```csh
sudo pkg update
sudo pkg upgrade -y
sudo pkg clean -y
```

### CUI アプリ

```csh
sudo pkg install -y zip unzip bzip2 \
    curl jq \
    git subversion vim
```


## アプリのインストール

### Python

* 本体

    ```csh
    sudo pkg install -y python39 py39-pip
    ```

### Go

```csh
sudo pkg install -y go
```

## 参考サイト

* pkg
    * [pkg](https://kaworu.jpn.org/freebsd/pkg)
