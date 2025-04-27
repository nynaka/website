[OCI SDK for Python](https://docs.oracle.com/ja-jp/iaas/Content/API/SDKDocs/pythonsdk.htm#SDK_for_Python)
===

## インストール

```bash
pip install oci
```

## [SDK の認証方法](https://docs.oracle.com/ja-jp/iaas/Content/API/Concepts/sdk_authentication_methods.htm)

認証方法は、いくつかあるようです。とりあえず、API キーベース認証の手順。

1. ユーザアカウントの選択

    **Identity & Security -> Domains -> <利用中のドメイン> -> Users** をたどり、API キーを設定するユーザアカウントを選択します。

2. **Resources** メニューの **API keys** リンクを押下します。
3. **Add API key** ボタンから API キーペアを生成、もしくは、既存の RSA 公開鍵のインポートを行います。キーペアを生成する場合、**Download private key** ボタンから秘密鍵をダウンロードしておきます。

4. 下記の形式でコンフィグファイルを作成する。

    ```text title="$HOME/.oci/config"
    [DEFAULT]
    user=ocid1.user.oc1..aaaaaaaal2chb6x7puy6ok4sqaeyx3kzls5vmy523mftriipacwnjcoqilmq
    fingerprint=f2:a7:88:50:ae:d0:c5:6c:d8:33:a4:48:58:42:f1:b5
    tenancy=ocid1.tenancy.oc1..aaaaaaaaq76m3fufrwtl7cor4jcfjpfjlacrfewxdu3wipcrtotdznsw62oq
    region=ap-tokyo-1
    key_file=<path to your private keyfile> # TODO
    ```

    コンフィグの設定方法については、[コードに書く方法](https://github.com/oracle/oci-python-sdk/blob/master/examples/configuration_example.py) もあるようです。

5. [oci-python-sdk の README.rst のコード](https://github.com/oracle/oci-python-sdk/blob/master/README.rst) を実行し、ユーザ情報が取得できれば、SDKの設定完了です。

