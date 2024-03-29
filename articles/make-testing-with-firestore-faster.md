---
title: "Firestore Emulator を使ったテストの実行時間を 20 倍以上短縮したぞ！"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - firebase
  - firestore
  - jest
  - nodejs
  - 高速化
published: true
---

## はじめに

みなさんは Firestore を使うことはありますか？

Firestore は低コストで利用できることから、個人開発者やスタートアップ企業の間でしばしば利用されることのあるデータベースです。

私が現在開発に関わっているサービスでも Firestore を利用しているのですが、構築初期段階と比較してテストにかかる時間が非常に長くなってきてしまっている問題が発生していました。

今回はその問題に対して `@firestore-emulator/jest` というライブラリを開発し、テストの実行時間を 20 倍以上短縮することに成功したので、その内容を紹介したいと思います。

## テストの実行時間が長くなってしまった問題

大前提として Firestore を使った場合のテスト方法はいくつかのパターンが存在します。

- GCP 上の実際の Firestore を利用する
- Firebase Local Emulator Suite を利用する
- Firestore のモックを利用する

これらのパターンにはそれぞれメリット・デメリットがあるのですが、私たちのチームでは再現性と信頼性のバランスを考えて、 Firebase の Emulator を用いてテストを実行していました。

しかし、 Firebase の Emulator はあまり使い勝手がいいものではありません。
特に致命的なのが Firestore などの起動ポートを cli から上書きできないことです。（もし上書きする方法をご存知の方いればぜひ教えてください）
一般的に jest などのテストランナーでは複数のテストファイルごとにプロセスを分けて実行することが多いのですが、そのプロセスごとに Firestore Emulator の起動ポートを変えられない都合上、並行するテストが 1 つの Firestore Emulator に対してアクセスを行うことになり、テスト間でデータを共有してしまう問題が発生してしまいます。
この問題は例えば `beforeEach` などの hook を用いて Firestore emulator の状態をリセットする際に、並行しているテストで利用するデータまで消してしまうことから、実質的に並行数を 1 にする必要があり、テストの実行時間が長くなってしまう原因となっていました。

この問題に対して、今回の現場では Docker を用いて Firebase Emulator をテストファイルごとに独立して起動させ、それぞれのコンテナに対し異なるポートを紐づけることで、テスト間でデータを共有することなくかつ並行してテストを実行できるようにしていました。

ただ、最近になってテストファイルが多くなるにつれ Docker を利用していることによるオーバーヘッドなどもあり、テストの実行時間が 10 分以上かかるようになってしまいました。
特に GitHub Actions 上でテストを実行していると、ローカルのマシンよりも低いスペックでテストを実行することから、計算資源不足でテストの実行自体も非常に不安定になってしまい、テストの信頼性が低下してしまうという問題も発生していました。

## Firestore Emulator を再発明する

そこで、私は Node.js で Firestore Emulator を再現するライブラリを開発することにしました。
特に今回のようにテストに Firestore を用いているような場合で、より便利にテストを実行できるようにすることをベースに設計を行いました。

実際に開発したライブラリは [`@firestore-emulator/jest`](https://www.npmjs.com/package/@firestore-emulator/jest) という名前で公開しています。

こだわった点としては、あくまで Firestore Emulator の代替を目指しているため、 firestore クライアントライブラリのモックを作るなどではなく、実際の Firestore と同じプロトコルでサーバーを実装しました。
幸いにも Firestore は gRPC で通信を行っており、 gRPC の proto ファイルは [`@google-cloud/firestore`](https://www.npmjs.com/package/@google-cloud/firestore) から得ることができたので、それに当てはまるように実装をすることで作っていくことができました。

このライブラリ自体は実際の Firestore Emulator を用いてテストを行なっているため、実装している範囲ではそれなりに高い精度で再現できていると考えています。

現時点で少なくとも [`firebase-admin`](https://www.npmjs.com/package/firebase-admin) でそれなりに動作できるようになっており、 CRUD だけではなく、 `onSnapshot` のようなリアルタイムリスナーにも対応しています。

## テストの実行時間を 20 倍以上短縮する

実際にこのライブラリを導入してみたのですが、プロダクションコードやテストコードの変更をほとんどすることなく、実行時間を 30 秒程度に短縮することに成功しました。元の実行時間が 10 分以上だったので、 20 倍以上の短縮になりました。

具体的な変更箇所としては、 `jest.config.js` ファイルや jest 用のセットアップファイルなどだけでしたので、既存の Firestore Emulator との互換性が高いことがわかります。

もし興味を持った方がいれば、具体的な利用法については npm のページに README を記載してあるので、そちらを参考に試してみてもらえればと思います！
もし未実装な部分や、動作しない箇所などあれば https://github.com/YutaUra/firestore-emulator の issue より報告していただけると幸いです！

## おわりに

僕はこれまで GraphQL が好きで、 GraphQL を使った開発をよくしていたのですが、一度 gRPC にも挑戦してみたいな〜とずっと思っていまして、今回思いがけない形で gRPC に入門することができて非常に楽しい経験をすることができました。

一応動作すると言っても、未実装な部分もありますし、 vitest のような他のテストランナーでも動作できるように拡充したいと考えているので、もし興味のある方いればぜひ issue で機能リクエストをしていただけると嬉しいです！
