---
title: "GraphQL の N+1 問題についての正しい解釈"
emoji: "🧩"
type: "tech"
topics: ["GraphQL"]
published: false
---

## はじめに

この記事では GraphQL の N+1 問題でよく言及されることについて、私なりの正しい解釈を提供することを目的としています。
GraphQL に精通している方にとっては特に新しい情報はないかもしれません。

主な対象読者は、 GraphQL を学習中の方や、 GraphQL における N+1 問題の本質を理解したい方です。

## N+1 問題とは

N+1 問題は、一般的にはデータベースへのクエリを繰り返し発行することで発生する問題として知られています。

例えば記事の一覧を取得し、それぞれの記事の著者名を取得するようなユースケースを考えてみましょう。

```javascript
function getArticlesWithAuthors() {
  const results = [];

  const articles = db.query(`SELECT * FROM articles`);

  for (const article of articles) {
    const author = db.query(`SELECT * FROM authors WHERE id = ?`, [
      article.authorId,
    ]);
    results.push({
      article: article,
      author: author,
    });
  }

  return results;
}
```

簡単な例ですが、上記のコードを実行すると、 `SELECT * FROM articles` は 1 回実行されて、取得した記事の数だけ `SELECT * FROM authors WHERE id = article.authorId` が実行されるため、合計で N+1 回のクエリが発行されます。
特に記事の数が多い場合、データベースへの負荷が増大し、パフォーマンスの低下を招く可能性があります。

このような問題を N+1 問題と呼びます。

N+1 問題では必要なデータを 1 回で取得することで、改善することができます。
特に RDBMS を利用するケースでは JOIN を利用することで、複数のテーブルから必要なデータを 1 回のクエリで取得することができます。

```javascript
function getArticlesWithAuthors() {
  const results = db.query(`
    SELECT
      articles.*,
      authors.*
    FROM articles
    JOIN authors ON
      articles.authorId = authors.id
  `);

  return results;
}
```

## GraphQL の N+1 問題とは

さて、GraphQL において N+1 問題はどのように発生するのでしょうか。
そのことについて説明を行う前に、 GraphQL がどのように動作するのかを簡単に説明します。

GraphQL サーバーは以下のようなリクエストを処理する必要があります。

```graphql
query getArticlesWithAuthors {
  articles {
    title
    author {
      name
    }
  }
}
```

この時、 GraphQL サーバーは以下のようなスキーマを持っているとします。

```graphql
type Query {
  articles: [Article]
}

type Article {
  title: String
  author: Author
}

type Author {
  name: String
}
```

このスキーマを満たす実装は以下のようになります。
GraphQL のリゾルバーの実装方法はいくつかありますが、最もシンプルな実装方法を示します。

```javascript
const resolvers = {
  Query: {
    articles: () => {
      return db.query(`SELECT * FROM articles`);
    },
  },
  Article: {
    author: (article) => {
      return db.query(`SELECT * FROM authors WHERE id = ?`, [article.authorId]);
    },
  },
};
```

この時、 GraphQL 実行エンジンは以下のような流れでリクエストを処理します。（あくまでイメージです）

```javascript
function execute(query, resolvers) {
  // 本当はクエリをパースして AST に変換したりとかしている
  // (省略)

  // Query.articles のリゾルバーを呼び出す
  const articles = resolvers.Query.articles().map((article) => {
    // article それぞれに対して Article.author のリゾルバーを呼び出す
    const author = resolvers.Article.author(article);
    return {
      ...article,
      author: author,
    };
  });

  return articles;
}
```

このように、 GraphQL ではリゾルバーが連鎖的に呼び出されることで、複雑なクエリを処理することができるようになっています。

また、上記の実装では `Article.author` のリゾルバーが記事の数だけ呼び出されるため、 N+1 問題が発生します。

特に GraphQL における N+1 問題とされた時は多くの場合、先程のようなことを指しているでしょう。

## GraphQL の N+1 問題は "問題" なのか

さて、前置きが長くなりましたが、ここで問題提起を行います。

私個人が観測する限り、 GraphQL の N+1 問題は "問題" として捉えられることが多いように感じます。

- 「GraphQL は N+1 問題が発生しやすい」
- 「GraphQL で N+1 問題を解決するためには DataLoader を使うべき」
- 「GraphQL で N+1 問題が発生するとパフォーマンスが低下する」

など、このような言及をよく見かける気がします。直接このような表現を行わなくとも、意味合いとして内心感じている方も多いかもしれません。

これらの言及は必ずしも間違っている訳ではありません。しかし、 GraphQL における N+1 問題について正しい解釈が行われているとも思えないのです。

## Web API における N+1 問題

ここで少し話をより抽象的に捉えてみましょう。

Web API において、 N+1 問題は大きく 2 つのタイプに分類することができると考えます。

1. 1 つのリクエストの中で N+1 問題が発生するタイプ
2. 複数のリクエストを通じて N+1 問題が発生するタイプ

これまで説明してきた N+1 問題は、前者に該当します。
しかし、実際には後者の方が問題として深刻であると私は考えています。

それはなぜか。REST API を用いた具体例を挙げて説明します。

**メディアサイトで記事一覧を表示する**

[zenn.dev](https://zenn.dev/) のようなメディアサイトを想定します。
トップページには複数の記事が表示されています。各記事には著者の名前やアイコンなども合わせて表示されているとします。

```javascript
const TopPage = () => {
  const articles = useFetch("/articles");

  return (
    <div>
      {articles.map((article) => (
        <Article article={article} />
      ))}
    </div>
  );
};

const Article = ({ article }) => {
  const author = useFetch(`/authors/${article.authorId}`);

  return (
    <div>
      <h2>{article.title}</h2>
      <div>
        <img src={author.iconUrl} />
        <span>{author.name}</span>
      </div>
    </div>
  );
};
```

例えば上記のような React コンポーネントを考えてみましょう。
このコンポーネントは REST API を用いて記事一覧を取得し、各記事に対して著者情報を取得して表示するものです。

この場合、トップページを表示するためには以下のリクエストが発行されます。

1. `/articles` へのリクエスト
2. 各記事に対して `/authors/:authorId` へのリクエスト

するとどうでしょうか、 `/articles` や `/authors/:authorId` の処理の中でそれぞれ 1 回ずつデータベース等へのアクセスが行われる場合、 N+1 問題が発生してしまうでしょう。

このケースは非常にわかりやすい例でしたが、複数のリクエストを通じて N+1 問題が発生するタイプの問題は、簡単に発生してしまう割に、問題を検知することが難しく、対処することも必ずしも簡単ではないという特徴があります。

## 複数のリクエストを通じて発生する N+1 問題の難しさ

複数のリクエストを通じて発生する N+1 問題の難しさとして、先程申し上げたように 2 つの特徴が挙げられます。

1. N+1 問題自体を検知することが難しい
2. N+1 問題を解決することが必ずしも簡単ではない

まずは 1 つ目の検知の難しさについて考えてみましょう。
理由は単純で、1 つ 1 つのリクエストでは N+1 問題が発生していないからです。
複数のリクエストを用いて 1 つの画面を作り上げるというユースケースがあって初めて問題が顕在化します。分散トレーシングの仕組みを用いていれば、1 つのユースケースで N+1 問題が発生することを検知することはできるかもしれませんが、 SPA やモバイルアプリを用いている場合に、ユースケースごとにリクエストを適切に分類して運用することは難しいでしょう。また、 Server Side Rendering や React Server Components などを用いる場合であっても、複数のリクエストを適切にまとめ上げて N+1 問題を検知することは簡単ではないはずです。

次に 2 つ目の解決の難しさについて考えてみましょう。
2 つ目の改題における解決策はいくつか存在します。例えば以下のような方法が考えられます。

- `/articles` の結果に著者情報を事前に含めておく
- `/authors/:authorId` の代わりに `/authors?ids=1,2,3` のようなバルクエンドポイントを用意する
