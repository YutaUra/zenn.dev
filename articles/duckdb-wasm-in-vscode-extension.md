---
title: "duckdb-wasm を使った VSCode 拡張機能の作り方"
emoji: "🦆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vscode", "duckdb", "wasm"]
published: true
---

この記事を必要とする人はあまりいないかもしれません。
VSCode の拡張機能を作る際に、 duckdb を利用するには癖があるためその解説を行いたいと思います。

## DuckDB とは

[DuckDB](https://duckdb.org/) とは、データ分析に特化した列指向データベースです。DuckDB は SQLite のように、埋め込み可能なデータベースエンジンとして設計されており、オフラインでの利用などを行う際に便利です。

DuckDB には [duckdb-wasm](https://github.com/duckdb/duckdb-wasm) という WebAssembly 版が存在し、ブラウザを含む様々な環境で DuckDB を利用することができます。

## モチベーション

VSCode の拡張機能として、SQL を interface とした構造的なファイル検索システムを作りたいと考えました。
イメージとしては、例えば github actions のワークフローが大量に存在していて、特定の条件に満たすワークフローを検索するときに、こんな SQL で検索できたら便利だなと思いませんか?

```sql
SELECT filename
FROM files
WHERE
  filename LIKE ".github/workflows/%.yml"
  AND (
    JSON_EXTRACT(content, "$.on.push.branches") IS NOT NULL
    OR JSON_EXTRACT(content, "$.on.pull_request.branches") IS NOT NULL
  )
```

なんか適当な感じですけど、何がいいたいかというと、 VSCode に組み込まれた検索機能では、このような複雑な検索は難しいです。
正規表現のエキスパートであれば達成可能かもしれませんが、 SQL で検索できたら、より柔軟な検索が可能になるのではないかと考えました。

そこで DuckDB を使って、ファイルの情報をデータベースとして保存することで、 SQL で検索することができる仕組みを作れないかと考えました。
一応他の手段としては、 SQLite を使うことも考えましたが、 DuckDB の方がイイ感じ(重要)な気がしたので、 DuckDB を頑張って使えるようにしました。

## duckdb-node と duckdb-wasm の違い

DuckDB には [duckdb-node](https://github.com/duckdb/duckdb-node) という Node.js 用のバインディングも存在します。

duckdb-node と duckdb-wasm の両方とも duckdb を利用するために使うことができるのですが、 duckdb-node を VSCode 拡張を使う際には2つの問題があります。

1.  duckdb-node を使う場合、 native モジュールを利用する必要があるため、ユーザーの環境に応じた native モジュールをバンドルさせる必要があります。
2.  native モジュールを利用するため、 VSCode Web Extension としては使えません。（多分）

特に 1. は、結構深刻ですよね。自分自身しか利用しないのであれば、特に問題はないと思うのですが、配布することを考えると、環境ごとの native モジュールをバンドルさせるのは少し現実的ではありません。

ということで、今回は duckdb-wasm を使って VSCode 拡張機能を作ることにしました。

## 結論

先に結論置いておきます。

1. duckdb-wasm の worker も単体でバンドルする
2. duckdb-wasm の wasm ファイルもアセットとしてバンドルする
3. duckdb-wasm の初期化処理も単体でバンドルする
4. [`web-worker`](https://www.npmjs.com/package/web-worker) を使って Node.js で duckdb-wasm の worker を利用する

基本的には以上だと思います。
詳しいバンドラの設定は最後に紹介しようと思いますが、上記の点を押さえておけばなんとかなると思います。

## duckdb-wasm の利用方法の紹介

基本的な使い方は duckdb-wasm の [examples](https://github.com/duckdb/duckdb-wasm/tree/a9a38bb480c1b44bb90e6fa35e731c572e8868ef/examples) が参考になるのですが、 VSCode 拡張機能として使う場合は少し工夫する必要があります。

大前提として、多くの場合 VSCode 拡張機能として作成する js ファイルはバンドルされることが多いです。
通常の npm ライブラリであれば、ユーザーがライブラリをインストールする際に合わせて依存関係もインストールもされるため、ライブラリ本体のコードのみを提供すれば問題ありませんが、 VSCode 拡張機能の場合は、ライブラリ本体と依存関係のあるライブラリ全てを配布する必要があります。その際に大量のファイルを配布するのはあまり嬉しいことではありません。そのため、 VSCode 拡張機能を開発する際は、最小限のファイルになるようにバンドルしたり、minify したりすることが一般的だと思います。

先ほどの `example` では Node.js での利用方法も紹介されていますが、バンドルされることまでは考慮されていません。バンドルされたとしても正しく動作するように構成する必要があります。

duckdb-wasm には以下の登場人物がいます。

- duckdb-wasm の wasm ファイル
- duckdb-wasm の worker 用の js ファイル
- duckdb-wasm のライブラリ本体

これらを組み合わせることで duckdb-wasm を利用することができます。具体的には以下のような手順で利用します。

```javascript
import path from 'node:path'
import duckdb from '@duckdb/duckdb-wasm'
import Worker from 'web-worker'

const DUCKDB_DIST = path.dirname(require.resolve('@duckdb/duckdb-wasm'));

const logger = new duckdb.ConsoleLogger();
const worker = new Worker(path.join(DUCKDB_DIST, "dist/duckdb-node-eh.worker.cjs"));
const db = new duckdb.AsyncDuckDB(logger, worker);
await db.instantiate(path.join(DUCKDB_DIST, "dist/duckdb-eh.wasm"));

const conn = await db.connect();
await conn.query(`SELECT count(*)::INTEGER as v FROM generate_series(0, 100) t(v)`);
```

通常であれば、上記のようにすることで期待通りに動作するはずです。しかし、バンドルをして、 VSCode 拡張機能として利用する場合は、以下のような問題が発生します。（厳密なエラーメッセージは忘れたので雰囲気です）

- `module not found @duckdb/duckdb-wasm` とか `module not found duckdb-node-eh.worker.cjs` エラーが発生する
- `module not found xxxx` エラーが発生する
  - `apache-arrow` など
- `wasm file not found` エラーが発生する
- `module not found vscode` エラーが発生する

1つずつ原因と対処方法を説明していきます。

### `module not found duckdb-node-eh.worker.cjs` エラーが発生する

これはバンドルされた後をイメージしてもらえたらわかりやすいと思います。

バンドル前のソースコードが以下のような構成になっているとします。

```text
.
|-- src
|   |-- extension.ts
|   |-- duckdb.ts
```

これを愚直にバンドルすると、以下のような構成になります。

```text
.
|-- dist
|   |-- extension.js
```

extension.ts と duckdb.ts がバンドルされて extension.js になっていますね。
この extension.js の中で `require.resolve('@duckdb/duckdb-wasm')` を実行するとどうなるでしょうか？

`@duckdb/duckdb-wasm` は node_modules にインストールされているライブラリですが、 `dist` フォルダには `node_modules` が存在せず、結果的に extension.js から `@duckdb/duckdb-wasm` を見つけることができません。

なので、 worker ファイルも合わせて `dist` フォルダに追加されるように設定する必要があります。

```text
.
|-- dist
|   |-- extension.js
|   |-- duckdb-node-eh.worker.cjs
```

```javascript
import path from 'node:path'
import duckdb from '@duckdb/duckdb-wasm'
import Worker from 'web-worker'

const logger = new duckdb.ConsoleLogger();
// dist フォルダにどのように配置されるのかイメージしつつパスを指定する
const worker = new Worker(path.join(__dirname, "duckdb-node-eh.worker.cjs"));
```

### `module not found xxxx` エラーが発生する

実は、単純に worker ファイルを dist フォルダにコピーするだけでは解決しません。

なぜなら、 worker ファイルでは `apache-arrow` などのライブラリが使用されており、それらも含めて worker ファイルにバンドルしてあげる必要あるからです。

なので、単純にコピーするのではなく、 entry point として `@duckdb/duckdb-wasm/dist/duckdb-node-eh.worker.cjs` を指定してあげてください。

### `wasm file not found` エラーが発生する

ここまで来ていれば、このエラーも解決できるはずです。

想像通り、 `.wasm` ファイルも dist フォルダーにコピーする必要があります。

バンドラの設定を用いて、 `.wasm` ファイルを dist フォルダにコピーするように設定してあげてください。

### `module not found vscode` エラーが発生することがある

そして、ここまで設定しても、なぜか `module not found vscode` エラーが発生することがあります。
そもそも `vscode` というモジュールは VSCode 拡張でしか利用できないモジュールで、裏を返せば `module not found vscode` エラーが発生することはないはずなので、非常に不思議です。

さて、かなり限られた情報しか提供していませんが、自信のある方は少し原因を考えてみましょう

---
---
---
---
---
---
---
---
---
---

はい。それでは正解の発表です。

正解は Worker(正確には `web-worker` による Node.js polyfill) として起動する際は Node.js の `worker_threads` が利用されます。子スレッドの実行コンテキストはメインスレッドとは異なるため、 `vscode` モジュールが利用できないということです。（詳しくは知りません。詳しい人いたら教えてください）

先述までのコードでは `vscode` モジュールは利用していませんが、実際の VSCode 拡張機能では以下のようなコードを書くことになります。

```javascript
import * as vscode from 'vscode'
import { initDB } from './duckdb' // 仮に duckdb の初期化処理を別ファイルに切り出している

export async function activate(context: vscode.ExtensionContext) {
  const db = await initDB()
  vscode.window.showInformationMessage('duckdb-wasm is activated!')
}
```

そうすると、バンドルされた `extension.js` はどうなるでしょうか?

```javascript
const vscode = require('vscode')

// @duckdb/duckdb-wasm
// ~~~~
// @duckdb/duckdb-wasm/dist/duckdb-node-eh.worker.cjs
// ~~~~
// web-worker
// ~~~~

// src/duckdb.ts
const initDB = async () => {
  // ...
  const worker = new Worker(path.join(__dirname, "duckdb-node-eh.worker.cjs"));
  // ...
}

// src/extension.ts
exports.activate = async function activate (context) {
  const db = await initDB()
  vscode.window.showInformationMessage('duckdb-wasm is activated!')
}
```

イメージですが、こんな感じになります。

すると worker_threads で起動される子スレッドは `extension.js` がベースとなってしまいます。 `extension.js` では `vscode` が読み込まれていますが、子スレッドでは `vscode` を利用することはできないため、 `module not found vscode` エラーが発生するというわけです。

そのため先述の `initDB` の処理は `extension.js` にバンドルさせずに、別ファイルに切り出す必要があります。

私自信 Node.js の worker_threads については、詳しくないため、別の方法で解決することができるかもしれませんが、私の知識ではこの方法しか思いつきませんでした。

## まとめ

ぐだぐだと書いてしまいましたが、私なりの結論をまとめたいと思います。

ファイル構成

```text
.
|-- dist
|   |-- extension.js
|   |-- duckdb.js
|   |-- duckdb-node-eh.worker.cjs
|   |-- duckdb-eh.wasm
|-- src
|   |-- extension.ts
|   |-- duckdb.ts
```

```typescript
// duckdb.ts
import { join } from "node:path";
import * as duckdb from "@duckdb/duckdb-wasm";
// bundler の loader で wasm ファイルを解決できるので、 import するだけで OK
import duckdb_wasm from "@duckdb/duckdb-wasm/dist/duckdb-eh.wasm";
import Worker from "web-worker";

export const initDb = async () => {
	const logger = new duckdb.ConsoleLogger();
	const worker = new Worker(
		new URL(`file://${join(__dirname, "./duckdb-node-eh.worker.cjs")}`),
	);
	const db = new duckdb.AsyncDuckDB(logger, worker);
	await db.instantiate(join(__dirname, duckdb_wasm));

	return db;
};
```

```typescript
// extension.ts
import * as vscode from "vscode";

// 動的読み込みするためのおまじない。（他にいい方法があれば教えてください）
const duckdb = require(`${"./duckdb"}`) as typeof import("./duckdb");
// webpack の場合はこちら
// const duckdb = __non_webpack_require__("./duckdb");

const dbPromise = duckdb.initDb();

export const activate = async (context: vscode.ExtensionContext) => {
  vscode.window.showInformationMessage("duckdb-wasm is activating...");
  const db = await dbPromise;
  vscode.window.showInformationMessage("duckdb-wasm is activated!");
};

export const deactivate = () => {
  const db = await dbPromise;
  await db.terminate();
};
```

続いてバンドラの設定です。

esbuild を使う場合

```json
{
  "entryPoints": {
    "extension": "src/extension.ts",
    "duckdb": "src/duckdb.ts",
    "duckdb-node-eh.worker": "node_modules/@duckdb/duckdb-wasm/dist/duckdb-node-eh.worker.cjs",
  },
  "outdir": "dist",
  "bundle": true,
  "format": "cjs",
  "platform": "node",
  "external": ["vscode"],
  "loader": {
    ".wasm": "file",
  }
}
```

なんとなくこんな感じの設定をすれば、動くのではないかと思います。

webpack を使う場合は、私の webpack 力が低くて嫌な感じになってしまうのですが、 `web-worker/node.js` と `@duckdb/duckdb-wasm/dist/duckdb-node.cjs` で利用されている dynamic require を dynamic require のまま処理させる方法がわからず、無理やり `__non_webpack_require__` に置き換えています。

そのために、 webpack や ts-loader などに加えて `string-replace-loader` も install してあげてください。

```javascript
const path = require('node:path');

/**@type {import('webpack').Configuration}*/
const config = {
  target: "node",
  entry: {
    extension: './src/extension.ts',
    _duckdb: './src/_duckdb.ts',
    duckdb_worker: './node_modules/@duckdb/duckdb-wasm/dist/duckdb-node-eh.worker.cjs'
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    libraryTarget: 'commonjs2',
    devtoolModuleFilenameTemplate: '../[resource-path]',
  },
  devtool: 'source-map',
  externals: {
    vscode: 'commonjs vscode'
  },
  resolve: {
    extensions: [".ts", ".js"],
  },
  module: {
    rules: [
      {
        // dynamic require のままにしたい
        test: [/node_modules\/web-worker\/node.js$/, /node_modules\/@duckdb\/duckdb-wasm\/dist\/duckdb-node.cjs$/],
        loader: 'string-replace-loader',
        options: {
          search: /require\((mod|s)\)/g,
          replace: '__non_webpack_require__($1)',
        },
      },
      {
        test: /\.ts$/,
        exclude: /node_modules/,
        use: [
          { loader: 'ts-loader' }
        ]
      },
      {
        test: /\.wasm$/,
        type: 'asset/resource'
      }
    ]
  },
};
module.exports = config;
```

あとは vite とか、そういうのでもいい感じにできるんじゃないでしょうか。

また、 VSCode Web Extension の場合は、 target の指定とか、 worker や wasm がまた変わってくるため、その辺りもいい感じに設定してみてください。

## おわりに

webpack で dynamic require をするとファイルが解決できない場合に `webpackEmptyContext` とかに変換されてエラーになっちゃうんですよね。runtime でファイルが存在していても関係ないって感じなので、いろいろ調べてみましたけど今回の方法しか見つけられませんでした。

esbuild の方がよっぽど素直なので、新規プロジェクトであれば esbuild を使うのがいいかもしれません。

というわけで、 duckdb-wasm を使った VSCode 拡張機能の作り方でした。
もし duckdb-wasm を VSCode 拡張に使ったよ！という方がいれば、ぜひ教えてください。
