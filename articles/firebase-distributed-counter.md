---
title: "Firebaseの拡張機能 Distributed Counter の使い方！"
emoji: "👍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["firebase"]
published: false
---

みなさん、 [Firebase](https://firebase.google.com/) は使っていますか？認証・ストレージ・DB が 1 つの SaaS として提供されているのでとても便利ですよね。

そして、そんな Firebase には拡張機能があるのをご存知ですか？存在自体は知っていても実際に利用したことがない方が大半ではないでしょうか？僕もその中の一人です。

今回はその中の [Distributed Counter(分散カウンター)](https://firebase.google.com/products/extensions/firestore-counter) を説明しつつ、実際に利用したりしてみたいと思います！

:::message info
Firebase の拡張機能は記事作成当時 β 版となっています。実際に利用される際はよく注意したり、ドキュメントを読むことを推奨します。
:::

# Distributed Counter(分散カウンター) とは

ここで、分散カウンターについて [Firebase の説明](https://firebase.google.com/docs/firestore/solutions/counters?hl=ja) を参考にしています。

:::message
以下、 https://firebase.google.com/docs/firestore/solutions/counters?hl=ja を参考にしています。実際に利用される際はこちらも合わせてご確認ください
:::

SNS によくある「いいね機能」ってありますよね。ユーザーが投稿に対して いいね をつけて、ユーザー全体のいいねの数を表示したりするという機能です。

実は Firebase 特に [Cloud Firestore(以下 Firestore)](https://firebase.google.com/docs/firestore) を使ってこの「いいね機能」を実装するのは意外と難しいんです。。

> Cloud Firestore では、1 つのドキュメントを 1 秒間に約 1 回しか更新できません。これはトラフィックの多いアプリケーションでは少なすぎる可能性があります。
> https://firebase.google.com/docs/firestore/solutions/counters?hl=ja

```
// ~~/posts/post-1
{
    title: ~~~,
    createdBy: ~~~,
    likeCount: 12 # ← このフィールドを更新するには制限がある。
}
```

つまり、複数のユーザーで 1 つの投稿にいいねをしたい場合、1 秒ずつしかいいねをすることができないということになります。

こういった場合の解決策の 1 つが 分散カウンター です。

分散カウンター がどのようにこの問題を解決しているかというと、いいねのカウントを「シャード」と呼ばれるサブコレクションに分解して保存するという方法です。

```
// ~~/posts/post-1
{
    title: ~~~,
    createdBy: ~~~,
    # likeCount: 12 いいね数はここでは扱わない
    shards: [
        ~~/posts/post-1/shards/1,
        ~~/posts/post-1/shards/2,
        ~~/posts/post-1/shards/3
    ] # 分割しているシャードを記録している。（暗黙的にどのシャードが使われるかわかる場合はなくても良いが、保存してあったほうが実装が楽かも）
}

# いいねの数を分割して保存する
// ~~/posts/post-1/shards/1
{
    count: 3
}
// ~~/posts/post-1/shards/2
{
    count: 5
}
// ~~/posts/post-1/shards/3
{
    count: 4
}
```

いいねをする場合は、これらのシャードからランダムに 1 つを選択して、それに対して更新の処理をするようにします。
また、いいねの合計を知りたい場合は全てのシャードを取得して、それらを合計するということでいいねの合計を知ることができるようになります。

今回の例であれば、複数のユーザーが同時にいいねをしても、1 秒間に 3 つまでは処理することができるということになります。（シャードが均等に選択されれば）

もちろん、これらには考慮すべきことがあって、

- シャードの数が少ない場合
  同時にいいねをする数に対して、シャードの数が少なければ、処理が追いつかず、再試行等が必要になってしまう場合があります。

- シャードの数が多すぎる場合
  いいねの合計を計算するために全てのシャードを取得する必要があるので、シャードの数が多くなればなるほど、ドキュメントの読み取り量が増えてしまうことになってしまい、コストが増加してしまいます。

# 分散カウンターの拡張機能について

分散カウンターは独自で実装することも可能ですが、 [拡張機能](https://firebase.google.com/products/extensions/) を用いることもできます。拡張機能は現在 β 版のもので、 [Cloud Functions for Firebase(以下 Functions)](https://firebase.google.com/docs/functions?hl=ja) などを必要とする機能をボタン 1 つでインストールすることで利用可能とするための機能です。

[分散カウンターの拡張機能](https://firebase.google.com/products/extensions/firestore-counter) のページからインストールすることができます。

詳しい内容は上記のリンクから確認していただきたいのですが、特に注意していただきたいことは

- Blaze プラン（従量課金プラン）である必要がある
- 全く使用されていないとしても、少額（通常は月額約$ 0.01）が請求される
- 3 つの Functions を作成します

:::message
記事の作成日（2021/08/06）での情報です。正確な内容はご自身でご確認ください
:::

# 使ってみた！

今回、僕が普段利用している Next.js をベースにサンプルを作成しますが、内容自体はフレームワーク関係ない内容となるように気遣っております。

## Firebase プロジェクトの作成

https://console.firebase.google.com/u/ から新しいプロジェクトを作成しましょう。

プロジェクト名はなんでもいいので、適当に作成してください。

プロジェクトが作成できたら

1. Firestore を有効化する

   1. 最初はとりあえずテストモードで作成します
   2. ロケーションは `asia-northeast-1`(東京) を選択します

2. 次に Blaze プランにアップグレードします。

   1. 下の画像のアップグレードを押す

      ![](https://storage.googleapis.com/zenn-user-upload/f24f68934b65b03c4ca5a296.png)

      :::message
      アップグレードすると、使用量に基づいて請求が来ます。いかなる責任も負うことができません
      :::

   2. 念の為予算アラートを設定しておく。僕はこういうサンプル系のプロジェクトでは $5 くらいを設定しています。

3. 拡張機能をインストールする

   1. [分散カウンターの拡張機能のページ](https://firebase.google.com/products/extensions/firestore-counter)を開く
   2. 「コンソールでインストールする」を押す

      :::message
      firebase-cli を使ってインストールすることもできま。
      :::

   3. 先程作成した Firebase プロジェクトを選択する
   4. 「有効な API と作成済みのリソースの確認」を確認して「次へ」を押す
   5. 「お支払いの情報と利用状況を確認」を確認して「次へ」を押す
   6. 「この拡張機能に付与されたアクセス権を確認」を確認して「次へ」を押す
   7. 「拡張機能を構成」で以下のように設定する

      1. Cloud Functions のロケーションを `Tokyo(asia-northeast-1)`(東京) にする。もしくは、 Firestore で設定したリージョンと同じにする
         :::message alert
         この項目は一度しか設定できません
         :::
      2. `Document path for internal state`と`Frequency for controllerCore function to be run`は後で変更ができるので、そのままにします。
      3. 「拡張機能をインストール」を押す

4. Web アプリの追加
   1. 「プロジェクトの概要」を開く
   2. Web を選択して、適当な名前をつけて作成する

以上で Firebase プロジェクトの作成は終了です！続いては実際のアプリを作成してみましょう。

## Web アプリの作成

全体のソースコード: https://github.com/YutaUra/firebase-distributed-counter-sample

### 分散カウンター用の設定をしていく

基本的にはこちらの [POSTINSTALL.md](https://github.com/firebase/extensions/blob/master/firestore-counter/POSTINSTALL.md) を参考にします。

#### 分散カウンターのクライアントコードを入手する

実は分散カウンターのクライアントコードは npm 等で配布されていないんですよね...。なので Github にあるものをコピーしてきます。

おそらく、 [ここ](https://github.com/firebase/extensions/blob/master/firestore-counter/clients/web/src/index.ts) にあると思うのですが、念の為上記の POSTINSTALL.md からリンクを拾ってきたほうが確実だと思います。

https://github.com/firebase/extensions/blob/master/firestore-counter/clients/web/src/index.ts

以下はこの記事作成時点でのソースコードです。上記リンクでやったけどうまく行かなかった場合などは下記コードと比較しながらうまいことやってみてください。

```ts
/*
 * Copyright 2019 Google LLC
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *    https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
import * as uuid from "uuid";

const SHARD_COLLECTION_ID = "_counter_shards_";
const COOKIE_NAME = "FIRESTORE_COUNTER_SHARD_ID";

export interface CounterSnapshot {
  exists: boolean;
  data: () => number;
}

export class Counter {
  private db: firebase.firestore.Firestore = null;
  private shardId = "";
  private shards: { [key: string]: number } = {};
  private notifyPromise: Promise<void> = null;

  /**
   * Constructs a sharded counter object that references to a field
   * in a document that is a counter.
   *
   * @param doc A reference to a document with a counter field.
   * @param field A path to a counter field in the above document.
   */
  constructor(
    private doc: firebase.firestore.DocumentReference,
    private field: string
  ) {
    this.db = doc.firestore;
    this.shardId = getShardId(COOKIE_NAME);

    const shardsRef = doc.collection(SHARD_COLLECTION_ID);
    this.shards[doc.path] = 0;
    this.shards[shardsRef.doc(this.shardId).path] = 0;
    this.shards[shardsRef.doc("\t" + this.shardId.substr(0, 4)).path] = 0;
    this.shards[shardsRef.doc("\t\t" + this.shardId.substr(0, 3)).path] = 0;
    this.shards[shardsRef.doc("\t\t\t" + this.shardId.substr(0, 2)).path] = 0;
    this.shards[shardsRef.doc("\t\t\t\t" + this.shardId.substr(0, 1)).path] = 0;
  }

  /**
   * Get latency compensated view of the counter.
   *
   * All local increments will be reflected in the counter even if the main
   * counter hasn't been updated yet.
   */
  public async get(options?: firebase.firestore.GetOptions): Promise<number> {
    const valuePromises = Object.keys(this.shards).map(async (path) => {
      const shard = await this.db.doc(path).get(options);
      return <number>shard.get(this.field) || 0;
    });
    const values = await Promise.all(valuePromises);
    return values.reduce((a, b) => a + b, 0);
  }

  /**
   * Listen to latency compensated view of the counter.
   *
   * All local increments to this counter will be immediately visible in the
   * snapshot.
   */
  public onSnapshot(observable: (next: CounterSnapshot) => void) {
    Object.keys(this.shards).forEach((path) => {
      this.db
        .doc(path)
        .onSnapshot((snap: firebase.firestore.DocumentSnapshot) => {
          this.shards[snap.ref.path] = snap.get(this.field) || 0;
          if (this.notifyPromise !== null) return;
          this.notifyPromise = schedule(() => {
            const sum = Object.values(this.shards).reduce((a, b) => a + b, 0);
            observable({
              exists: true,
              data: () => sum,
            });
            this.notifyPromise = null;
          });
        });
    });
  }

  /**
   * Increment the counter by a given value.
   *
   * e.g.
   * const counter = new sharded.Counter(db.doc("path/document"), "counter");
   * counter.incrementBy(1);
   */
  public incrementBy(val: number): Promise<void> {
    // @ts-ignore
    const increment: any = firebase.firestore.FieldValue.increment(val);
    const update: { [key: string]: any } = this.field
      .split(".")
      .reverse()
      .reduce((value, name) => ({ [name]: value }), increment);
    return this.doc
      .collection(SHARD_COLLECTION_ID)
      .doc(this.shardId)
      .set(update, { merge: true });
  }

  /**
   * Access the assigned shard directly. Useful to update multiple counters
   * at the same time, batches or transactions.
   *
   * e.g.
   * const counter = new sharded.Counter(db.doc("path/counter"), "");
   * const shardRef = counter.shard();
   * shardRef.set({"counter1", firestore.FieldValue.Increment(1),
   *               "counter2", firestore.FieldValue.Increment(1));
   */
  public shard(): firebase.firestore.DocumentReference {
    return this.doc.collection(SHARD_COLLECTION_ID).doc(this.shardId);
  }
}

async function schedule<T>(func: () => T): Promise<T> {
  return new Promise<T>(async (resolve) => {
    setTimeout(async () => {
      const result = func();
      resolve(result);
    }, 0);
  });
}

function getShardId(cookie: string): string {
  const result = new RegExp(
    "(?:^|; )" + encodeURIComponent(cookie) + "=([^;]*)"
  ).exec(document.cookie);
  if (result) return result[1];

  const shardId = uuid.v4();

  const date = new Date();
  date.setTime(date.getTime() + 30 * 24 * 60 * 60 * 1000);
  const expires = "; expires=" + date.toUTCString();

  document.cookie =
    encodeURIComponent(cookie) + "=" + shardId + expires + "; path=/";
  return shardId;
}
```

そして、ファイルを作成し、上記の内容を丸ごとコピペします。

一部以下のように変更します。

```ts
import firebase from "firebase"; // import を追加

// ...

export class Counter {
  private db: firebase.firestore.Firestore; // = null を削除
  private shardId = "";
  private shards: { [key: string]: number } = {};
  private notifyPromise: null | Promise<void> = null; // null | Promise<void>に変更
  // ....

  /**
   * Listen to latency compensated view of the counter.
   *
   * All local increments to this counter will be immediately visible in the
   * snapshot.
   */
  public onSnapshot(observable: (next: CounterSnapshot) => void) {
    // 購読終了用の関数を返す
    const unsubscribes = Object.keys(this.shards).map((path) => {
      return this.db
        .doc(path)
        .onSnapshot((snap: firebase.firestore.DocumentSnapshot) => {
          this.shards[snap.ref.path] = snap.get(this.field) || 0;
          if (this.notifyPromise !== null) return;
          this.notifyPromise = schedule(() => {
            const sum = Object.values(this.shards).reduce((a, b) => a + b, 0);
            observable({
              exists: true,
              data: () => sum,
            });
            this.notifyPromise = null;
          });
        });
    });
    return () => {
      unsubscribes.forEach((unsubscribe) => {
        unsubscribe();
      });
    };
  }
}
```
