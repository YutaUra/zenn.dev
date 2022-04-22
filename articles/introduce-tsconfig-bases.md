---
title: "tsconfig/bases の紹介！"
emoji: "🪐"
type: "tech"
topics: ["typescript"]
published: true
---

## 結論

```sh
$ yarn add -D @tsconfig/strictest
```

```json:tsconfig.json
{
    "extends": "@tsconfig/strictest/tsconfig.json",
}
```

以上です！

## `tsconfig.json` ってどんなふうに書いていますか？？

`tsconfig.json` をこんな感じで書いている人はいないでしょうか

```json:tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "allowUnusedLabels": false,
        "allowUnreachableCode": false,
        "exactOptionalPropertyTypes": true,
        "noFallthroughCasesInSwitch": true,
        "noImplicitOverride": true,
        "noImplicitReturns": true,
        "noPropertyAccessFromIndexSignature": true,
        "noUncheckedIndexedAccess": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "lib": ["dom"],
        "outDir": "./dist",
        // ...以下略
    }
}
```

そして、これを複数のプロジェクトでコピペして使い回したり。。。

まぁ、全然別にいいと思うんですが、こんなものもあるよ！っていうのを紹介させてください！

## tsconfig/bases で `tsconfig.json` を簡潔にする

結局僕らって `"strict": true` にするし、 `"noUncheckedIndexedAccess": true` にするじゃないですか、もっというと型が厳しくなればなるほど喜んじゃうド M じゃないですか（）

そういう「とりあえず堅くしておきたい！」っていう設定をデフォルトにしてくれる便利なやつがあるんですよ。

https://github.com/tsconfig/bases

こいつです！

細かいものについては [README](https://github.com/tsconfig/bases#readme) を見てもらえるのが良いのですが、 [@tsconfig/strictest](https://www.npmjs.com/package/@tsconfig/strictest) を使うと、型チェックを堅くするオプションを全部乗せてくれるんですよね。

使い方は普通にインストールして、

```sh
$ yarn add -D @tsconfig/strictest
```

`tsconfig.json` の `extends` に指定するだけです

```json:tsconfig.json
{
    "extends": "@tsconfig/strictest/tsconfig.json",
}
```

めっちゃ簡単でしょ？

これを指定すると、以下の内容が適用されるようになるんです。

https://github.com/tsconfig/bases/blob/27f9097a02f960e351d69bca8dd4c7581d70e1b6/bases/strictest.json

僕らのチームには厳しすぎるなぁっていうルールがあれば、それだけを明示的にオフにしてあげれば良いんですよね。

ということで、ぜひ使ってみてください！

## 番外編: "importsNotUsedAsValues": "error"

`strictest` には `"importsNotUsedAsValues": "error"` というオプションがあるんですが、これは型情報のためだけに `import` している場合に `import type` として `import` させなければならないというルールです。具体的に言うと、

```ts
// これはエラーになる
import { Foo } from "./foo";

const foo: Foo = {};

// この場合正しいのは
import type { Foo } from "./foo";
```

```ts
// これはエラーにならない
import { Foo } from "./foo";

const foo = new Foo();
```

という感じなのですが、いちいち手動で管理なんかやってられないですよね。実は `@typescript-eslint/eslint-plugin` のルールにこれを自動で修正してくれるルールがあるので、それを設定すると良いです。

```json:.eslintrc
{
    "rules": {
        "@typescript-eslint/consistent-type-imports": "error",
    }
}
```

また、このルールは `@typescript-eslint/recommended` という設定にも含まれているので、上記のように指定するか

```json:.eslintrc
{
    "extends": ["plugin:@typescript-eslint/recommended"]
}
```

のようにしても大丈夫です。
