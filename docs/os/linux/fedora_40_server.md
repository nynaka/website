Fedora Server
===

!> VMware Fusion にインストールすると dnf コマンドの応答が強烈に遅い。  
動き出せば普通なので、コマンド実行から応答処理が始まるまでの間に何かあるっぽい。

## 設定

* /etc/fstab

    mount オプションに noatime,nodiratime を追加すると、ファイルやディレクトリのアクセス時間が、アクセスされる度に更新されなくなるため、若干、ファイルアクセスが速くなるらしい。

    ```text:/etc/fstab
    UUID=22748c10-a83b-490d-802d-dc50fcefe72a /                       xfs     defaults        0 0
    ↓
    UUID=22748c10-a83b-490d-802d-dc50fcefe72a /                       xfs     noatime,nodiratime        0 0
    ```

## アプリのインストール

### サーバ関連

* インストール

    ```bash
    sudo dnf install -y openssh avahi
    ```

* 起動設定

    ```bash
    sudo systemctl enable sshd avahi-daemon
    sudo systemctl start sshd avahi-daemon
    ```

### CLI アプリ

* 主要ツール

    ```bash
    sudo dnf install -y \
        sudo zip unzip bzip2 p7zip \
        git subversion \
        vim curl wget jq
    ```

* man コマンド

    ```bash
    sudo dnf install -y glibc-langpack-ja man
    ```

### 言語

* C/C++

    ```bash
    sudo dnf groupinstall "Development Tools" -y
    sudo dnf install -y g++ cmake gdb
    ```

* C#

    ```bash
    sudo dnf install -y mono-devel
    ```

    * [Download - Stable | Mono](https://www.mono-project.com/download/stable/#download-lin-ubuntu)

* [Kotlin](https://kotlinlang.org/docs/command-line.html)

    dnf リポジトリには Kotlin のコマンドラインコンパイラは登録されていないので、sdkman か snap を利用してインストールします。

    * [sdkman のインストール](https://sdkman.io/install)

        ```bash
        curl -s "https://get.sdkman.io" | bash
        source "$HOME/.sdkman/bin/sdkman-init.sh"
        ```

    * Kotlin のインストール

        ```bash
        sdk install kotlin
        ```

* Java

    ```bash
    sudo dnf install -y java-latest-openjdk-devel
    ```

* Python

    ```bash
    sudo dnf install -y python3 python3-pip
    ```

    * 開発支援ツール

        ```bash
        pip3 install --upgrade --break-system-packages \
            bandit flake8-bandit \
            isort mypy ruff
        ```

     * Web 開発

        ```bash
        pip3 install --upgrade --break-system-packages \
            flask django
        ```

* Go

    ```bash
    sudo dnf install -y golang
    ```

* Rust

    ```bash
    sudo dnf install -y rust
    ```


### VMware Tools

* インストール

    ```bash
    sudo dnf install -y open-vm-tools
    sudo systemctl start vmtoolsd
    sudo systemctl enable vmtoolsd
    ```

* マウント設定

    * マウントポイントの作成

        ```bash
        sudo mkdir -p /mnt/vbox_share
        ```

    * /etc/fstab に追加

        ```bash
        vbox_share  /mnt/vbox_share vboxsf  defaults  0  0
        ```

    * マウントの実行

        ```bash
        sudo systemctl daemon-reload
        sudo mount -a
        ```


### [Install Docker Engine on Fedora](https://docs.docker.com/engine/install/fedora/)

1. 古いバージョンの削除

    ```bash
    sudo dnf remove docker \
            docker-client \
            docker-client-latest \
            docker-common \
            docker-latest \
            docker-latest-logrotate \
            docker-logrotate \
            docker-selinux \
            docker-engine-selinux \
            docker-engine
    ```

2. dnf リポジトリの設定

    ```bash
    sudo dnf -y install dnf-plugins-core
    sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
    ```

3. Docker (関連ツール含む) のインストール

    ```bash
    sudo dnf update
    sudo dnf install -y \
        docker-ce \
        docker-ce-cli \
        containerd.io \
        docker-buildx-plugin \
        docker-compose-plugin
    ```

4. 権限設定

    ```bash
    sudo usermod -aG docker fedora
    ```

5. その他

    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```
