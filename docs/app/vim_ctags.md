Vim タグジャンプ
===

## 言語

### Python

* ツールのインストール

    * Debian 系 Linux

        ```bash
        sudo apt install -y exuberant-ctags
        ```

* ctags が対応している言語・拡張子の確認

    ```bash
    ctags --list-maps
    ```

* Python コードだけを対象としたタグ生成

    ```bash
    ctags --languages=Python -R .
    ```


## タグジャンプ

| コマンド              | 内容                                                             |
| :-------------------- | :--------------------------------------------------------------- |
| Ctrl-]                | カーソルのある単語に対してタグジャンプ                           |
| [N]Ctrl-t             | N 個前のタグ位置に戻る (タグスタックは保持)                      |
| :[N]pop(:po)          | N 個前のタグ位置に戻る (タグスタックを削除)                      |
| :tags                 | タグスタックを表示                                               |
| tag(:ta) keyword      | キーワードの定義位置にタグジャンプする                           |
| :tselect(:ts) keyword | キーワードのジャンプ先候補を表示                                 |
| :tjump(:tj) keyword   | キーワードのジャンプ先候補を表示 (1個しかない場合は直接ジャンプ) |
| :tnext(:tn)           | 次の候補へジャンプ                                               |
| :tNext(:tN)           | 前の候補へジャンプ                                               |
| :tprevious(:tp)       | 前の候補へジャンプ                                               |
| :trewind(:tr)         | 最初に一致した候補へジャンプ                                     |
| :tfirst(:tf)          | 最初に一致した候補へジャンプ                                     |
| :tlast                | 最後に一致した候補へジャンプ                                     |
| g]                    | カーソル位置の単語に対して :tselect を実行                       |
| g Ctrl-]              | カーソル位置の単語に対して :tjump を実行                         |


## 参考サイト

* [とほほのVim入門(タグジャンプ)](https://www.tohoho-web.com/vim/tag.html)
* [タグジャンプを実現するためにctagsの環境を整える](https://ryamashina.com/itml/20210429/)
