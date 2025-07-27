Hello Cython
===

## Cython のインストール

```bash
pip install cython
```


## プロジェクトの構成

```text
hello_cython/
├ hello.pyx         # Cythonコード
├ setup.py          # ビルドスクリプト
└ main.py           # 呼び出し用のPythonコード
```

- hello.pyx

    ```cython
    def say_hello():
        print("Hello Cython")
    ```

- setup.py

    ```bash
    from setuptools import setup
    from Cython.Build import cythonize

    setup(
        ext_modules=cythonize("hello.pyx")
    )
    ```

- main.py

    ```python
    import hello

    hello.say_hello()
    ```


## 実行方法

1. ビルドツールのインストール

    - Debian / Ubuntu

        ```bash
        build-essential libpython3-dev
        ```

2. cython のインストール

    ```bash
    pip install cython
    ```

3. ビルド

    ```bash
    python setup.py build_ext --inplace
    ```

4. main.py の実行

    ```bash
    python main.py
    ```


## その他

### 生成する拡張モジュール名を指定する

setup.py で拡張モジュール名やバージョンを設定することができます。

```python
from setuptools import setup, Extension
from Cython.Build import cythonize

extensions = [
    Extension(
        name="my_hello",           # モジュール名（importするときの名前）
        sources=["hello.pyx"],     # 元ファイル（.pyx）
    )
]

setup(
    name="hello_cython_project",
    version="0.0.1",
    ext_modules=cythonize(extensions),
)
```

これでビルドすると拡張モジュール名は **my_hello** になります。  
main.py は下記のように修正します。

```python
import my_hello

my_hello.say_hello()
```
