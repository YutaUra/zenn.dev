---
title: "async/awaitネイティブのためのちょっとしたPromiseの使い方"
emoji: "🧚‍♂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript"]
published: true
---

皆さんは Promise を使っていますか？ ES2017 で async/await が導入されてから、直接 Promise を使う場面は減ったのではないでしょうか？

かくいう私も、2019 年からプログラミングを始めているので、最初から async/await がある世界で、直接 Promise を使う機会はあまり多くはありませんでした。

この記事はそんな async/await ネイティブのためのちょっとした Promise の使い方を解説していきたいと思います。（もし間違って解説しちゃってたら教えてください 🙏）

:::message
`Promise.all` などは使いますが、 `new Promise` はあんまり使わないですよね
:::

## async/await を使ったコードと Promise を使ったコード

まずは、 Promise って何なのか、どのように使うのか、 使い慣れた async/await と比較しながら解説して行きます。

まずは、最も簡単な async/await の使用方法を例に解説して行きます。

```ts
async function foo() {
  const res = await bar();
  console.log(res);
}
```

この関数を Promise を使って書くと以下のようになります。

```ts
function foo() {
  return bar().then((res) => {
    return console.log(res);
  });
}
```

おっと、、 Promise を使うと言っていたくせに `Promise` が使われていません。。
ではどこに `Promise` が使われているのでしょうか。

実は `bar` 関数が返している値というのが **Promise オブジェクト** になっているのです。

```ts
function foo() {
  const promise = bar(); // promise は Promise オブジェクトとなっている
  return promise.then((res) => {
    return console.log(res);
  });
}
```

つまり、 Promise とは **Promise オブジェクト** を使った非同期処理のパターンだということになります。
**Promise オブジェクト** についてはこの後解説していきます。

また、少しそれますが Promise がなかった頃に使われていた callback パターンではどうなるのかも合わせて紹介します。

```ts
// 先述までの foo, bar 関数とは互換性がないので注意
function foo(callback) {
  bar((res) => {
    console.log(res);
    callback();
  });
}
```

callback パターンでは、実行したい処理を callback 関数として受け渡すことで非同期処理を実現しているということになります。
callback パターンは async/await 記法と比べると非常にわかりにくいですね。

:::message
Promise も callback パターンの一つだと考えることもできますが、メソッドチェーンや、例外処理などの点で callback パターンとは異なっていると言えます。
:::

## Promise オブジェクト

ここでは簡単に Promise オブジェクトについて解説していきます。
詳しい解説が必要であれば、 [プロミスの使用 - JavaScript | MDN](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises)を参照してもらえればと思います。

### Promise オブジェクトの作成

Promise オブジェクトは `Promise` コンストラクタ関数を使って作成することができます。（`Promise.all` などについては省略）

```ts
const promise = new Promise((resolve, reject) => {
  // 非同期処理
});
```

`Promise` コンストラクタ関数は `resolve` と `reject` の 2 つの引数に取る関数を引数に取ります。
そして、 `Promise` コンストラクタ関数の内部で結果が取得できた際は `resolve` 関数を呼び出し、エラーが発生した場合は `reject` 関数を呼び出すことで、結果を返すことができます。

```ts
const promise = new Promise((resolve, reject) => {
  let error, result;
  // 何かしらの処理

  if (error) {
    // エラーが発生した場合は、reject にエラー内容を渡す
    reject(error);
  } else {
    // 結果が取得できた場合は、resolve に結果を渡す
    resolve(result);
  }
});
```

また、 `resolve` と `reject` は複数回実行しても、どちらか最初に実行したもののみが有効になります。

```ts
const promise = new Promise((resolve, reject) => {
  let error, result;
  // 何かしらの処理

  reject(error);

  // ここの resolve は無視される
  resolve(result);
  // 同様に reject も無視される
  reject(error);
});
```

また、`Promise` コンストラクタ関数には async 関数を渡すこともできるので、こういうこともできます。

```ts
const promise = new Promise(async (resolve, reject) => {
  try {
    const result = await foo();
    resolve(result);
  } catch (error) {
    reject(error);
  }
});
```

### Promise オブジェクトから結果を取得する

Promise オブジェクトから結果を取得するには `then` メソッドを使います。
また、`catch` メソッドを使うとエラーが発生した場合にエラー内容を取得することができ、 `finally` メソッドを使うと非同期処理が成功または失敗した後に処理を行うことができます。

```ts
promise
  .then((res) => {
    // res には promise が resolve した際の値が入っている
    console.log(res);
  })
  .catch((err) => {
    // err には promise が reject した際の値が入っている
    console.error(err);
  })
  .finally(() => {
    // finally は成功または失敗した後に実行される
    console.log("finished");
  });
```

[`then` メソッドの連鎖](https://developer.mozilla.org/ja/docs/Web/JavaScript/Guide/Using_promises#%E9%80%A3%E9%8E%96) などについても知っておくと良いかと思いますが、ここでは省略します。

---

ここまでで、 **Promise オブジェクト** について 作成方法 と 値の受け取り方について解説してきました。
これ以降では、目的であるちょっとした使い方を紹介して行きます。

## ちょっとした Promise の使い方

- [ポーリングをしながら、非同期処理を実行する](#ポーリングをしながら非同期処理を実行する)
- [TypeScript での Promise.all と filter](#TypeScript%20での%20Promise.all%20と%20filter)

### ポーリングをしながら非同期処理を実行する

これはどちらかというと、 `setInterval` を `Promise` 的に扱う方法になってしまうのですが、ポーリングが必要な場面を Promise で扱いたい場合には、この方法が使えます。

例えば Firebase を採用している際に、

1. クライアントサイドで Firebase Authentication を利用してユーザーの登録を行う
2. ユーザーの登録をトリガーに Cloud Function で Firestore にユーザー情報を保存する
3. クライアントサイドでは作成されるユーザー情報を使用したいので、Cloud Function の実行を待ちたい

こういう要件あるのではないでしょうか？Firebase を使っていない場合でも、イベント駆動アーキテクチャを採用していると、こういう場面は多いかと思います。

こういう場合に async/await を使う場合このようになります。

```ts
const createUser = async (data) => {
  // Firebase Authentication などの ユーザーを作成する関数
  const authUser = await auth.createUser(data);

  const interval = setInterval(async () => {
    // Firestore などの DB からユーザー情報を取得する関数
    const user = await db.getUser(authUser.id);

    if (user) {
      // ユーザー情報が取得できたら、ポーリングを終了する
      clearInterval(interval);
      // !!! return して user を返したいが、setInterval の中なので createUser としては返すことができない
      return user;
    }
  }, 1000 /* 仮に1秒ごとにポーリングする */);

  return user; // ???
};
```

こういう場合は Promise を使ってあげましょう。

```ts
const createUser = async (data) => {
  const authUser = await auth.createUser(data);

  return new Promise(async (resolve, reject) => {
    const interval = setInterval(() => {
      const user = await db.getUser(authUser.id);
      if (user) {
        clearInterval(interval);
        resolve(user);
      }
    }, 1000);
  });
};
```

どうでしょうか？これで `createUser` 関数はサーバーサイドでの処理を待って値を返すことができるようになりました。

ただ一応、async/await でも `setInterval` ではなく、 `while` を使っても書くこともできて、

```ts
// msミリ秒待つ関数
const wait = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

const createUser = async (data) => {
  const authUser = await auth.createUser(data);

  let flag = true;
  let ret = null;

  while (flag) {
    user = await db.getUser(authUser.id);
    if (user) {
      flag = false;
      ret = user;
    }

    await wait(1000);
  }
  return ret;
};
```

個人的にはこちらで書くよりも、`setInterval` を使う方が良さそうに見えます。

### TypeScript での Promise.all と filter

ここの内容は、

簡単に `Promise.all` を説明します。
`Promise.all` は 複数の Promise オブジェクトを並列で実行し、それら全てが成功する（またはどれかが失敗する）のを待って結果を返す関数です。

```ts
const [res1, res2] = await Promise.all([fetch1(), fetch2()]);
```

`Promise.all` は 配列を引数に取る性質から、 `Array.map` を合わせて使う場面も多いです。

```ts
const results = await Promise.all(
  items.map(async (item) => {
    const res1 = await fetch1(item.id);
    return fetch2(res1.id);
  })
);
```

`Promise.all` と `Array.map` を合わせて使う場合で、特定の条件で処理をスキップすると言った場合、

```ts
const results = await Promise.all(
  items.map(async (item) => {
    if (!item.public) {
      // スキップしたい
      return null;
    }
    const res1 = await fetch1(item.id);
    return fetch2(res1.id);
  })
);
```

このように書くことができるのですが、この場合、 `results` は `(typeof item | null)[]` になってしまいます。なので、もちろん

```ts
// 型ガード関数を使う
const isNotNull = <T>(item: T): item is Exclude<T, null> => item !== null;

const results = (
  await Promise.all(
    items.map(async (item) => {
      if (!item.public) {
        // スキップしたい
        return null;
      }
      const res1 = await fetch1(item.id);
      return fetch2(res1.id);
    })
  )
).filter(isNotNull);
```

としても良いのですが、ちょっと汚いですよね。できれば `Promise.all` の中で `filter` したいです。

```ts
const isNotNull = <T>(item: T): item is Exclude<T, null> => item !== null;

const results = await Promise.all(
  items
    .map(async (item) => {
      if (!item.public) {
        // スキップしたい
        return null;
      }
      const res1 = await fetch1(item.id);
      return fetch2(res1.id);
    })
    .filter(isNotNull)
);
```

しかし、これはうまく行きません。

なぜなら、`items.map` が返しているは Promise オブジェクトの配列だからです。 Promise オブジェクトは null ではないため、フィルターされることはありません。その後 `Promise.all` によって `null` という結果が `results` に入ってしまいます。

これは、 `async` 関数は必ず Promise オブジェクトを返す という当たり前の仕様によって引き起こされています。
なので、

```ts
const isNotNull = <T>(item: T): item is Exclude<T, null> => item !== null;

const results = await Promise.all(
  items
    .map((item) => {
      if (!item.public) {
        // スキップしたい
        return null;
      }
      return fetch1(item.id).then((res1) => fetch2(res1.id));
    })
    .filter(isNotNull)
);
```

このように、 `items.map` 関数が Promise オブジェクトまたは null を返すようにしてあげることで、 filter 関数を機能させることができるようになります。

## おわりに

記事を書いた後、これは正直どうでもいいことかもなぁとか思ったのですがせっかくなので公開することにしました。
誰かの助けになれば幸いです。

他にも async/await ではなく Promise を使わなければいけない場面とかあれば是非教えてほしいです。
