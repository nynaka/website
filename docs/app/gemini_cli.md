[Gemini CLI](https://cloud.google.com/blog/ja/topics/developers-practitioners/introducing-gemini-cli/)
===

## [インストール](https://github.com/google-gemini/gemini-cli?tab=readme-ov-file#-installation)

### Node.js のインストール

- nvm のインストール

    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
    ```

- .bashrc の再読み込み

    ```bash
    source $HOME/.bashrc
    ```

- nvm でインストールできる Node.js のバージョン確認

    ```bash
    nvm ls-remote --lts
    ```

- Node.js のインストール (ここでは v24.12.0)

    ```bash
    nvm install v24.12.0
    ```

- バージョンの確認

    ```basb
    node -v
    npm -v
    npx -v
    ```

### [Gemini CLI](https://github.com/google-gemini/gemini-cli?tab=readme-ov-file#-installation)

```bash
npm install -g @google/gemini-cli
```


## 参考サイト

- [https://codelabs.developers.google.com/gemini-cli-hands-on?hl=ja#0](https://codelabs.developers.google.com/gemini-cli-hands-on?hl=ja#0)