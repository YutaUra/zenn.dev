---
title: "エンジニアにとってのatomicデザイン"
emoji: "👨🏿‍🎤"
type: "idea"
topics: [design]
published: false
---

## atomic デザイン？

atomic デザインをご存知でしょうか

僕は atomic デザインと聞くと以下のフォルダ構成のことを思い浮かべます

```
/root
  └ /src
      └ /components
      │  ├ /atoms
      │  │   ├ Button.tsx
      │  │   └ Heading.tsx
      │  ├ /molecules
      │  │   ├ TextareaField.tsx
      │  │   └ SearchField.tsx
      │  ├ /organisms
      │  │   ├ Header.tsx
      │  │   └ HeroSection.tsx
      │  └ /templates
      │      ├ SignInTemplate.tsx
      │      └ DefaultTemplate.tsx
      └ /pages
         ├ index.tsx
         └ signin.tsx

```

atomic デザインを採用しているとチームごとに多少差はあれど、何となくこのような構成になることは多いのではないでしょうか？

## atomic デザイン論争

## atomic デザインは何を解決する？

そもそも atomic デザインとは何なのでしょうか？

## エンジニアから見たときの atomic デザインについて考える

## 結論

僕はエンジニアにとっての atomic デザインは **責任の分離** 、 **状態管理** などをルール化する際の手助けにするべきものなのかなと思いました。

「atomic デザインが本当はこういうものだ」とか、「atoms か molecules か organisms のどれに入れるべきかわからない」というのは本来意味のない話ではないでしょうか？エンジニアにとって重要なのは、適切にコンポーネントを分けることによって得られる再利用性の向上・責任の分離であるはずです。

この記事をきっかけにいろいろなチームで atomic デザインのあり方について考えてもらえれば嬉しいなと思います。

また、今回では atomic デザインという言葉を使いましたが、エンジニアにとっての atomic デザイン専用の言葉もあるといいのかなと思うのですが、どなたかご存知ですか？
