Node.js のインストール
===

## [nvm](https://github.com/nvm-sh/nvm) のインストール

```bash
curl -o- \
    https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh \
    | bash
```

インストールが完了すると、

```bash
=> Compressing and cleaning up git repository

=> Appending nvm source string to /home/ubuntu/.bashrc
=> Appending bash_completion source string to /home/ubuntu/.bashrc
=> Close and reopen your terminal to start using nvm or run the following to use it now:

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

のようなメッセージが表示され、$home/.bashrc に最後の 3 行が追加されていると思います。この後、

```bash
source $HOME/.bashrc
```

を実行すると nvm コマンドが使えるようになると思います。


## 主な nvm サブコマンド

- nvm ls-remote/nvm list-remote

    nvm コマンドでインストールできる Node.js のバージョンリストを表示します。

- nvm ls/nvm list

    自分の環境にインストールしている Node.js のバージョンリストを表示します。

- nvm install <version>

    指定したバージョンの Node.js をインストールします。

    ```bash title="実行例"
    nvm install v22.17.1
    ```
