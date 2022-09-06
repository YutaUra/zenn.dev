---
title: "Schema Stitching と Apollo Federation の比較"
emoji: "⚖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - graphql
  - microservices
published: true
---

:::message
YouTube でも解説してます！動画で視聴したい方は是非 YouTube もご利用ください。
:::

https://www.youtube.com/watch?v=RY8W38gppo4


## GraphQL とマイクロサービス

GraphQL を利用してマイクロサービスを作成する方法として代表的なものに Schema Stitching と Apollo Federation v2（以下 Apollo Federation） がある。

この2つが達成したいことは基本的には同じではあるものの、アプローチの仕方などには差がある。

この記事の中では Schema Stitching と Apollo Federation をなるべくフラットに比較し、それぞれにどういった違いがあるのかについて説明をしていく。

## 共通のコンセプト

Schema Stitching にしても Apollo Federation にしても共通している部分が存在している。

それはスキーマのマージである。

![https://www.graphql-tools.com/assets/distributed-graph.png](https://www.graphql-tools.com/assets/distributed-graph.png)

スキーマのマージがどのように行われているのか、具体的な例を持って説明を行う

### スキーマのマージ

```tsx
import { makeExecutableSchema } from '@graphql-tools/schema';
import { stitchSchemas } from '@graphql-tools/stitch';

let postsSchema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type Post {
      id: ID!
      text: String
      userId: ID!
    }

    type Query {
      postById(id: ID!): Post
      postsByUserId(userId: ID!): [Post]!
    }
  `,
  resolvers: { ... }
});

let usersSchema = makeExecutableSchema({
  typeDefs: /* GraphQL */ `
    type User {
      id: ID!
      email: String
    }

    type Query {
      userById(id: ID!): User
    }
  `,
  resolvers: { ... }
});

// setup subschema configurations
export const postsSubschema = { schema: postsSchema };
export const usersSubschema = { schema: usersSchema };

// build the combined schema
export const gatewaySchema = stitchSchemas({
  subschemas: [
    postsSubschema,
    usersSubschema,
  ]
});
```

```graphql
type Query {
  postById(id: ID!): Post
  postsByUserId(userId: ID!): [Post]!
  userById(id: ID!): User
}
```

ここでのスキーマはローカルで定義されているものに感じますが、ローカルのスキーマかリモートのスキーマかはあまり重要ではありません。

重要なのは、 `postsSchema` と `usersSchema` がマージされているということです。

 `postById` クエリは `postsSchema` へのリクエストとなり、 `userById` クエリは `usersSchema` へのクエリへと自動的に呼び分けてくれます。

```graphql
query Hoge {
  postById(id: "") {
    id
    text
  }
  userById(id: "") {
    id
    email
  }
}
```

```graphql
query _postsSchema {
  postById(id: "") {
    id
    text
  }
}

query _usersSchema {
  userById(id: "") {
    id
    email
  }
}
```

それぞれのスキーマが解決できるようにクエリを自動的に分割し、解決させた後に結果を合体させる。スキーマのマージによって、本来複数のサービスに対してリクエストを行わなければならないものを、一つのリクエストで一度に取ることができるようになった。

ここまでは Schema Stitching と Apollo Federation で大きな差はない。大きく違うのは *Schema 間のフィールド解決* である。

### Schema 間のフィールド解決とは

言葉だけではあまりイメージしにくいと思うので、ここでは簡単に Schema 間のフィールド解決について説明を行うことにする。

結論をいうと、別 Schema に存在するタイプをフィールドに持ちたいといったような場合のことである。

具体的なイメージとしては

```graphql
"""
usersSchema
"""
type User {
  id: ID!
  email: String
  posts: Post[] # ここで postsSchema のタイプを参照している
}

"""
postsSchema
"""
type Post {
  id: ID!
  text: String
  userId: ID!
  user: User # ここで usersSchema のタイプを参照している。
}
```

基本的に `usersSchema` 単体では `Post` を解決することはできないし、 `postsSchema` も `User` を解決することはできない。

サービスを跨いだ解決は Gateway 層を絡めて解決する必要がある。

次からそれぞれでどのように Schema 間のフィールド解決を行っているのかを説明する。

## Schema Stitching

[Schema Stitching](https://www.graphql-tools.com/docs/schema-stitching/stitch-combining-schemas) は graphql-tools によって提供される機能の一つである。

Schema Stitching でのフィールド解決のコンセプトはデリゲートです。

デリゲートとはフィールド解決のロジックを他の Schema に委譲するということです。

```tsx
import { stitchSchemas } from '@graphql-tools/stitch'
import { delegateToSchema } from '@graphql-tools/delegate'

export const schema = stitchSchemas({
  subschemas: [postsSubschema, usersSubschema],
  typeDefs: /* GraphQL */ `
    extend type Post {
      user: User! # postsSchema には存在しないフィールド
    }
    extend type User {
      posts: [Post!]! # usersSchema には存在しないフィールド
    }
  `,
  resolvers: {
    User: {
      posts: {
        selectionSet: `{ id }`,
        resolve(user, args, context, info) {
          // ここで usersSchema から postsSchema へ委譲を行う
          return delegateToSchema({
            schema: postsSubschema,
            operation: 'query',
            fieldName: 'postsByUserId',
            args: { userId: user.id },
            context,
            info
          })
        }
      }
    },
    Post: {
      user: {
        selectionSet: `{ userId }`,
        resolve(post, args, context, info) {
          // ここで postsSchema から usersSchema へ委譲を行う
          return delegateToSchema({
            schema: usersSubschema,
            operation: 'query',
            fieldName: 'userById',
            args: { id: post.userId },
            context,
            info
          })
        }
      }
    }
  }
})
```

このように記載されているとき、

```graphql
query Hoge {
  postById(id: "") {
    id
    text
    user {
      id
      email
      posts {
        id
        text
      }
    }
  }
}
```

次のようなクエリに展開され順次解決されていくようなイメージとなる。

```graphql
query _postsSchema1 {
  postById(id: "") {
    id
    text
    userId # selectionSet によって追加される
  }
}

query _usersSchema {
  userById(id: _postsSchema1.postById.userId) {
    id
    email
  }
}

query _postsSchema2 {
  postsByUserId(id: _usersSchema.userById.id) {
    id
    text
  }
}
```

ここで、 `Post.user` を解決するためには `Post.userId` が必要なため、 `postById` クエリに `userId` フィールドがない場合に自動的に追加されるように `selectionSet` オプションを用いている。

Schema Stitching での Schema 間のフィールド解決は比較的単純だが、やや冗長となっていることがわかるだろう。

それに対して Apollo Federation ではまた違ったアプローチをしているので、次で詳しく説明を行う。

## Apollo Federation v2

ここでは Apollo Federation v1 については説明を行わないのでご了承ください。

Apollo Federation での基本的なコンセプトは タイプの分離から関心の分離 を行うということです。

タイプの分離とはどういうことかというと以下のように subgraph を分けることです。

```graphql
# Users subgraph
type User {
  id: ID!
  name: String
  reviews: [Review]
  purchases: [Product]
}
```

```graphql
# Products subgraph
type Product {
  id: ID!
  name: String
  price: String
  reviews: [Review]
}
```

```graphql
# Reviews subgraph
type Review {
  id: ID!
  body: String
  author: User
  product: Product
}
```

それに対して、関心の分離を行うとは以下のように subgraph を分けることです。

```graphql
# Users subgraph
type User {
  id: ID!
  name: String
}
```

```graphql
# Products subgraph
type Product {
  id: ID!
  name: String
  price: String
}

type User {
  id: ID!
  purchases: [Product]
}
```

```graphql
# Reviews subgraph
type Review {
  id: ID!
  body: String
  author: User
  product: Product
}

type User {
  id: ID!
  reviews: [Review]
}

type Product {
  id: ID!
  reviews: [Review]
}
```

これによって各 subgraph での関心が適切に分離できていることがわかります。

タイプによる分離は本質的には Schema Stitching と同じであり、 Schema 間の解決を行う場合に Gateway が管理をしなければならないことは明白です。ということは Schema 間の関連が多くなれば Gateway は肥大化していき、巨大なグラフを効果的に築くことが難しくなるのはいうまでもありません。

それでは、Apollo Federation がどのように subgraph 間のフィールドを解決するのかを解説します。

Apollo Federation では `@key` ディレクティブを用いてタイプの定義を行います。

```graphql
# Products subgraph
type Product @key(fields: "id") {
  id: ID!
  name: String!
  price: Int
}
```

```graphql
# Reviews subgraph
type Product @key(fields: "id", resolvable: false) {
  id: ID!
}

type Review {
  score: Int!
  description: String!
  product: Product!
}

```

`Product` タイプは Products subgraph で保存・管理されているデータです。

そして、 Products subgraph では `Product.__resolveReference` フィールドを実装する必要があります。

```tsx
// Products subgraph
const resolvers = {
  Product: {
    __resolveReference(productRepresentation) {
      return fetchProductByID(productRepresentation.id);
    }
  },
}
```

このように `__resolveReference` を実装してあげることで、自動的に他 subgraph にあるタイプを解決できるようになります。

```graphql
query GetReviewsWithProducts {
  latestReviews { # Defined in Reviews
    score
    product {
      id
      price # ⚠️ NOT defined in Reviews!
    }
  }
}
```

このクエリが以下のように展開されるイメージです。

```graphql
query GetReviewsWithProducts {
  latestReviews {
    score
    product {
      __typename # __typename を使うために自動的に追加されます。
      id
    }
  }
}

query {
  # このクエリで Reviews subgraph だけでは取得できないフィールドを取得できるようにしている。
  _entities(representations: [{"__typename": "Product", "id": "1"}, {"__typename": "Product", "id": "2"}]) {
    ... on Product {
      price
    }
  }
}
```

各 subgraph では `_entities` というクエリが利用できるようになっていて、 `representations` の `__typename` をみて `Product` ならば `Product.__resolveReference` から Product データを取得できるようになっている。

## それぞれのメリットやデメリットの比較

Schema Stitching でも Apollo Federation でも複数の Schema, subgraph を跨いだデータ取得が可能になっている。そして、一概にどちらか一方が正しいとは言えないが、メリットとデメリットを個人的な主観に基づいて述べさてもらう。

### Schema Stitching のメリット

- それぞれの Schema にたいして特別な何かをする必要はない。
    - 例えば GitHub API などの既存の GraphQL サーバーに対しても簡単に結合を行うことができる。

### Schema Stitching のデメリット

- Gateway が肥大化してしまう。
    - 性質上 Schema を跨いだフィールドは Gateway に記述する必要があり、Gateway を薄く保つことはかなり難しい。

### Apollo Federation のメリット

- subgraph 間のフィールド解決は `@key` ディレクティブによって自動的に解決されるため、Gateway をかなり薄くすることができる。
- subgraph ごとにそれぞれ関心ごとを独立することができる

### Apollo Federation のデメリット

- Apollo Federation では GitHub API などのサービスは一度 subgraph としてラップしてあげる必要があり、 Apollo Federation の仕様に束縛されてしまう
- NestJS や TypeGraphQL でも Apollo Federation のサポートがされているものの、基本的に Apollo のオレオレ感は否めない。
    - Apollo と心中する覚悟が必要かも?

## まとめ

Apollo Federation には独自の仕様があり、クセはあるものの、その分恩恵も大きい。

現時点では、なるべく GraphQL 標準でシンプルを維持するには Schema Stitching がベストである。

マイクロサービスに関する問題は Apollo Federation や Schema Stitching に関することだけではなく、デプロイの問題や障害点の問題などいくつか考えられる。実際に運用してみて得られた知見などがあれば共有したいと思う。

## 参考記事へのリンク

[https://www.graphql-tools.com/docs/schema-stitching/stitch-combining-schemas](https://www.graphql-tools.com/docs/schema-stitching/stitch-combining-schemas)

[https://www.apollographql.com/docs/federation/v2](https://www.apollographql.com/docs/federation/v2)