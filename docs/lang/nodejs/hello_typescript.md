Hello Typescript
===

## TypeScript の実行

1. ワークディレクトリの作成

    ```bash
    mkdir -p ./typescript/hello_typescript
    cd ./typescript/hello_typescript
    ```

2. プロジェクトの初期化

    ```bash
    npm init -y
    ```

3. typescript のインストール

    ```bash
    npm install --save-dev typescript
    ```

4. typescript のソース準備

    ```ts title="hello_typescript.ts"
    const message: string = "Hello TypeScript";
    console.log(message);
    ```

5. typescript コードのコンパイル

    npx は node package execute の略で、typescript のソースコードを javascript に変換してくれます。

    ```bash
    npx tsc hello_typescript.ts
    ```

6. javascript の実行

    ```bash
    node hello_typescript.js
    ```

## TypeScript のコンパイルを省略する

1. ts-node のインストール

    ```bash
    npm install --save-dev ts-node
    ```

2. tsconfig.json 作成

    ```bash
    npx tsc --init
    ```

3. ts-node を使って hello_typescript.ts の実行

    ```bash
    npx ts-node hello_typescript.ts
    ```
