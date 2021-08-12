---
title: "正しく使う ReactContext"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: true
---

みなさん、 `React` の `Context` は正しく使えていますか？この記事ではパフォーマンスの観点で `Context` を少しでも正しく使うための方法や理由などを書いていこうと思います。

なお、この記事の内容が最も正しいと主張するつもりではありません。ぜひ PR や コメント でよりより使い方を共有してください！

# 想定する読者と記事の範囲

一番この記事を読んでいただきたいのはこういった方々です

- `Context` についてなんとなくしか分かっていない
- とりあえず `redux` や `recoil` 等を使えば良いと思っている

しばしば `recoil` と `Context` を比較するといった趣旨の記事があったりしますが、 `Context` について正しく使えていないが故に、適切に比較できないものがあったりします。僕自身は `Context` よりも `recoil` を使うことが多いのですが、思考停止で `recoil` を使うのでは無く、なぜ `recoil` が嬉しいのかなどを正しく理解することで、より `React` を好きになってもらえるように解説したいと思いました。

この記事の範囲は以下の通りです

- `Context` の正しい使い方
- `recoil` との正しい比較方法

`Context` についてより深く知りたいと思っている方にはぜひ読んでいただきたいです。

# `Context` のアンチパターン

`Context` の正しい使い方を説明するために、アンチパターンについて説明したいと思います。
`Context` を紹介している記事などでよくみられるアンチパターンは主に以下の 2 通りがあります。

1. 値本体と値を入れる関数を 1 の `Context` に入れている
2. `Provider` の中に子コンポーネントを書いている

これらについて、具体的な例を挙げた上でなぜアンチパターンなのか解説していきます。

## アンチパターン 1. 値本体と値を入れる関数を 1 の `Context` に入れている

まずはアンチパターンの具体的な例を示したいと思います。

```tsx
import {
  createContext,
  Dispatch,
  FC,
  SetStateAction,
  useCallback,
  useContext,
  useState,
} from "react";
import "../App.css";

interface CountState {
  count: number;
  setCount: Dispatch<SetStateAction<number>>;
}

const countContext = createContext<CountState>({
  count: 0,
  setCount: () => undefined,
});

const CountProvider: FC = ({ children }) => {
  const [count, setCount] = useState<number>(0);

  return (
    <countContext.Provider value={{ count, setCount }}>
      {children}
    </countContext.Provider>
  );
};

const useCountValue = () => useContext(countContext).count;
const useCountSetValue = () => useContext(countContext).setCount;

const Button = () => {
  console.log("render ボタン");

  const setCount = useCountSetValue();

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, [setCount]);

  return (
    <div>
      <button onClick={increment}>+1</button>
    </div>
  );
};

const DisplayCount = () => {
  console.log("render カウント");

  const count = useCountValue();

  return <p>カウント: {count}</p>;
};

const OtherComponent = () => {
  console.log("render 全然関係ないコンポーネント");

  return <p>全然関係ない</p>;
};

const App = () => {
  return (
    <div className="App">
      <DisplayCount />
      <Button />
      <OtherComponent />
    </div>
  );
};

const Root = () => {
  return (
    <CountProvider>
      <App />
    </CountProvider>
  );
};

export default Root;
```

こういった書き方をしている記事をわりと見かけます。この書き方がなぜダメか、実際に動かして確認してみます。

![](https://storage.googleapis.com/zenn-user-upload/aa2a2ae328c36ab3e7ef581c.gif)

もちろん、 `Context` を利用しているので値をバケツリレーせずに取得・変更することができていますが、カウントの値が変わるたびにボタンまでレンダリングされています。
しかし、ボタンはカウントの値に関係ないので、本来レンダリングは不要なはずです。

原因はカウントの値と変更するための関数が 1 つのオブジェクトとしてプロバイドしているところにあります。このようにしてしまうと、 `setCount` 関数自体は毎回同じでも、 `count` が変わるごとに新しいオブジェクトが生成されてしまうので、 `count` に依存していないボタンコンポーネントもそれに引きづられてレンダリングされてしまうということになります。

なので、以下のように値と変更するための関数でそれぞれ `Context` を作ってあげてください。

```tsx
import {
  createContext,
  Dispatch,
  FC,
  SetStateAction,
  useCallback,
  useContext,
  useState,
} from "react";
import "../App.css";

// Context を二つに分ける
const countContext = createContext<number>(0);
const setCountContext = createContext<Dispatch<SetStateAction<number>>>(
  () => undefined
);

const CountProvider: FC = ({ children }) => {
  const [count, setCount] = useState<number>(0);

  return (
    <countContext.Provider value={count}>
      <setCountContext.Provider value={setCount}>
        {children}
      </setCountContext.Provider>
    </countContext.Provider>
  );
};

const useCountValue = () => useContext(countContext);
const useCountSetValue = () => useContext(setCountContext);

const Button = () => {
  console.log("render ボタン");

  const setCount = useCountSetValue();

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, [setCount]);

  return (
    <div>
      <button onClick={increment}>+1</button>
    </div>
  );
};

const DisplayCount = () => {
  console.log("render カウント");

  const count = useCountValue();

  return <p>カウント: {count}</p>;
};

const OtherComponent = () => {
  console.log("render 全然関係ないコンポーネント");

  return <p>全然関係ない</p>;
};

const App = () => {
  return (
    <div className="App">
      <DisplayCount />
      <Button />
      <OtherComponent />
    </div>
  );
};

const Root = () => {
  return (
    <CountProvider>
      <App />
    </CountProvider>
  );
};

export default Root;
```

実際に動かしてみると

![](https://storage.googleapis.com/zenn-user-upload/afe738d485fa6df0ed002a81.gif)

このように、カウント部分のみがレンダリングされているということがわかります。

## アンチパターン 2. `Provider` の中に子コンポーネントを書いている

まずは具体例を書いていきます。

```tsx
import {
  createContext,
  Dispatch,
  SetStateAction,
  useCallback,
  useContext,
  useState,
} from "react";
import "../App.css";

const countContext = createContext<number>(0);
const setCountContext = createContext<Dispatch<SetStateAction<number>>>(
  () => undefined
);

const useCountValue = () => useContext(countContext);
const useCountSetValue = () => useContext(setCountContext);

const Button = () => {
  console.log("render ボタン");

  const setCount = useCountSetValue();

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, [setCount]);

  return (
    <div>
      <button onClick={increment}>+1</button>
    </div>
  );
};

const DisplayCount = () => {
  console.log("render カウント");

  const count = useCountValue();

  return <p>カウント: {count}</p>;
};

const OtherComponent = () => {
  console.log("render 全然関係ないコンポーネント");

  return <p>全然関係ない</p>;
};

const App = () => {
  console.log("render App");

  return (
    <div className="App">
      <DisplayCount />
      <Button />
      <OtherComponent />
    </div>
  );
};

const Root = () => {
  const [count, setCount] = useState<number>(0);

  return (
    <countContext.Provider value={count}>
      <setCountContext.Provider value={setCount}>
        <App />
      </setCountContext.Provider>
    </countContext.Provider>
  );
};

export default Root;
```

上記のように、 `Root` 関数内で `Provider` と `App` を呼び出しています。この場合に実行するとどのようになるのでしょうか。

![](https://storage.googleapis.com/zenn-user-upload/4ea0858bd318a8d89e8d2175.gif)

すると、 ボタンコンポーネントもそうですが、全然関係ないコンポーネントも App コンポーネントも全てレンダリングされてしまいます。

よく考えてみれば当たり前だったりするのですが、こういう書き方で紹介されることもあるので解説していきます。

`Context` も基本的には `Provider` 部分で `useState` などで確保した値を使っているに過ぎません。なので、 `Provider` に子コンポーネントを書いてしまうと、値が変わるたびに全てレンダリングされ直してしまいます。

子コンポーネントは `props` として受け取りましょう。

```tsx
import {
  createContext,
  Dispatch,
  FC,
  SetStateAction,
  useCallback,
  useContext,
  useState,
} from "react";
import "../App.css";

const countContext = createContext<number>(0);
const setCountContext = createContext<Dispatch<SetStateAction<number>>>(
  () => undefined
);

// Provider は props で子コンポーネントを受ける
const CountProvider: FC = ({ children }) => {
  const [count, setCount] = useState<number>(0);

  return (
    <countContext.Provider value={count}>
      <setCountContext.Provider value={setCount}>
        {children}
      </setCountContext.Provider>
    </countContext.Provider>
  );
};

const useCountValue = () => useContext(countContext);
const useCountSetValue = () => useContext(setCountContext);

const Button = () => {
  console.log("render ボタン");

  const setCount = useCountSetValue();

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, [setCount]);

  return (
    <div>
      <button onClick={increment}>+1</button>
    </div>
  );
};

const DisplayCount = () => {
  console.log("render カウント");

  const count = useCountValue();

  return <p>カウント: {count}</p>;
};

const OtherComponent = () => {
  console.log("render 全然関係ないコンポーネント");

  return <p>全然関係ない</p>;
};

const App = () => {
  console.log("render App");

  return (
    <div className="App">
      <DisplayCount />
      <Button />
      <OtherComponent />
    </div>
  );
};

const Root = () => {
  return (
    <CountProvider>
      <App />
    </CountProvider>
  );
};

export default Root;
```

このように、 `Provider` が `props` で子コンポーネントを受け取るとどうなるでしょうか

![](https://storage.googleapis.com/zenn-user-upload/41241cccde0126638fe2070d.gif)

このように、 `App` コンポーネントなどはレンダリングされなくなります。

## 正しい `Context` の使い方

これまでのアンチパターンを踏まえて正しい `Context` の使い方をお見せします。

```tsx
import {
  createContext,
  Dispatch,
  FC,
  SetStateAction,
  useContext,
  useState,
} from "react";
import "../App.css";

// コンテキストは 値・値を入れる関数 で分けて作る
const countContext = createContext<number>(0);
const setCountContext = createContext<Dispatch<SetStateAction<number>>>(
  () => undefined
);

// Provider は props で子コンポーネントを受ける
const CountProvider: FC = ({ children }) => {
  const [count, setCount] = useState<number>(0);

  return (
    <countContext.Provider value={count}>
      <setCountContext.Provider value={setCount}>
        {children}
      </setCountContext.Provider>
    </countContext.Provider>
  );
};

const useCountValue = () => useContext(countContext);
const useCountSetValue = () => useContext(setCountContext);
```

以上が正しい `Context` の使い方となります。

# recoil との比較

最後に recoil と使用方法を比べてみたいと思います。

以下のコードで比べてみたいと思います。

```tsx
import {
  createContext,
  Dispatch,
  FC,
  SetStateAction,
  useCallback,
  useContext,
  useState,
} from "react";
import { atom, RecoilRoot, useRecoilValue, useSetRecoilState } from "recoil";
import "../App.css";

const countAtom = atom<number>({
  key: "countAtom",
  default: 0,
});

const countContext = createContext<number>(0);
const setCountContext = createContext<Dispatch<SetStateAction<number>>>(
  () => undefined
);

const CountProvider: FC = ({ children }) => {
  const [count, setCount] = useState<number>(0);

  return (
    <countContext.Provider value={count}>
      <setCountContext.Provider value={setCount}>
        {children}
      </setCountContext.Provider>
    </countContext.Provider>
  );
};

const useCountValue = () => useContext(countContext);
const useCountSetValue = () => useContext(setCountContext);
const useRecoilCountValue = () => useRecoilValue(countAtom);
const useRecoilCountSetValue = () => useSetRecoilState(countAtom);

const Button = () => {
  console.log("render ボタン");

  const setCount = useCountSetValue();

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, [setCount]);

  return (
    <div>
      <button onClick={increment}>+1</button>
    </div>
  );
};

const DisplayCount = () => {
  console.log("render カウント");

  const count = useCountValue();

  return <p>カウント: {count}</p>;
};

const RecoilButton = () => {
  console.log("render Recoil ボタン");

  const setCount = useRecoilCountSetValue();

  const increment = useCallback(() => {
    setCount((prev) => prev + 1);
  }, [setCount]);

  return (
    <div>
      <button onClick={increment}>Recoil +1</button>
    </div>
  );
};

const RecoilDisplayCount = () => {
  console.log("render Recoil カウント");

  const count = useRecoilCountValue();

  return <p>Recoil カウント: {count}</p>;
};

const OtherComponent = () => {
  console.log("render 全然関係ないコンポーネント");

  return <p>全然関係ない</p>;
};

const App = () => {
  return (
    <div className="App">
      <DisplayCount />
      <Button />
      <RecoilDisplayCount />
      <RecoilButton />
      <OtherComponent />
    </div>
  );
};

const Root = () => {
  return (
    <CountProvider>
      <RecoilRoot>
        <App />
      </RecoilRoot>
    </CountProvider>
  );
};

export default Root;
```

実際に動かしてみるとこのように、最小限のレンダリングとなっているということがわかります。

![](https://storage.googleapis.com/zenn-user-upload/d41a7bdf9158413dd117977b.gif)

純粋にレンダリング回数のみの比較ですが、 `Context` を使っても recoil を使っても同じように最適化することができるということがわかりました。

その結果を踏まえて recoil の嬉しいところが何かを考えてみるとそれは **圧倒的に記述量が減る** ことなのかなと思います。

`Context` を使う場合、1 つの状態を扱うのに値・値を入れる関数の 2 つのコンテキストが必要で、 `useState` を利用した `Provider` も必要となります。

recoil を使うと、 `atom` を宣言するだけでグローバルな状態を作成することができます。圧倒的に記述する量を減らすことができますよね。

また、もう少し込み入った場合では `Map` 形式で値を扱いたい場合があります。そういった場合に recoil では `atomFamily` といった関数が用意されており、簡単に最適化することができます。

# おわりに

最後は少しそれて recoil の話になりましたが、この記事を通して `Context` について理解をすることができたでしょうか？

もしわからないことなどあればコメントで聞いてください！
