Windows Tips
===

## Windows サービスの再起動

1. 管理者モードで Powershell を起動する。
2. Restart-Service コマンドで対象のサービスを再起動する。

    ```powershell title="Hyper-V ホストコンピューティングサービスの例"
    Restart-Service -name vmcompute
    ```

    WSL 起動時に認証っぽいエラーで終了する場合、このサービスを再起動することで改善することがあります。ただ、このサービスを再起動することで Windows が異常終了し、青画面に遭遇することもあります。

## Java 関連

- 環境変数

    | 変数      |            値             |
    | :-------- | :-----------------------: |
    | JAVA_HOME | C:\tool\oracle_jdk-21.0.4 |
    | Path      |      %JAVA_HOME%\bin      |

- 環境変数 JAVA_HOME のコマンド設定方法

    - bin フォルダの親フォルダまでのフルパスを設定する。
    - 管理者モードでコマンドプロンプトを追加し、下記のようにコマンドを実行する。

        ```cmd
        setx /m JAVA_HOME "C:\tool\oracle_jdk-21.0.4"
        ```
