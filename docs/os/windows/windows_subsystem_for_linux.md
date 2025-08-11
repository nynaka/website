Windows Subsystem for Linux (WSL)
===

## インストール

### 前提条件

|     |                                      |
| :-- | :----------------------------------- |
| CPU | x86_64                               |
| OS  | Windows 11 Pro（バージョン21H2以降） |

### インストール手順

1. 管理者権限で Powershell 起動
2. WSL のインストール

    ```powershell
    wsl --install
    ```

    このコマンドで WSL2 と既定の Linux ディストリビューション（Ubuntu）がインストールされます。  
    Ubuntu 24.04 をインストールしたい場合は、下記のように Linux ディストリビューションを指定してインストールコマンドを実行します。

    ```powershell
    wsl --install -d Ubuntu-24.04
    ```

3. Windows の再起動
4. WSL ゲスト OS の初期設定

    OS 再起動後、WSL 起動時にユーザー名とパスワードを設定します。

    ```bash
    Enter new UNIX username: [ユーザー名を入力]
    Enter new UNIX password: [パスワードを入力]
    Retype new UNIX password: [パスワードを再入力]
    ```

---

## wsl コマンド

- wsl コマンドでインストールできる Linux 一覧表示

    ```bash
    wsl --list --online
    # または
    wsl -l -o
    ```

- WSL 環境にインストールした Linux 一覧表示

    ```bash
    wsl --list --verbose
    # または
    wsl -l -v
    ```

- WSL 環境で起動している Linux の停止

    ```bash
    wsl --terminate Ubuntu-24.04
    # または
    wsl -t Ubuntu-24.04
    ```

- WSL 環境で起動しているすべての Linux の停止

    ```bash
    wsl --shutdown
    ```



