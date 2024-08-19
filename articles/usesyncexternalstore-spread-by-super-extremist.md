---
title: "超過激派が布教する！ useSyncExternalStore の使い方"
emoji: "😈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

この記事は [uhyo](https://zenn.dev/uhyo) さんの [過激派が教える！　useEffectの正しい使い方](https://zenn.dev/uhyo/articles/useeffect-taught-by-extremist) という記事に触発され執筆されました。まだ読まれていない方は一度読まれることをお勧めします。

https://zenn.dev/uhyo/articles/useeffect-taught-by-extremist

前提として、この記事は uhyo さんによる記事に付け加えて超過激派の人や超過激派になりたいという人向けの記事となっており、非常に刺激が強いためそうではない方は uhyo さんによる記事の内容までに留めることをお勧めします（？）

## 基本原則

React では動的な値をリアクティブに画面に反映させるための仕組みとして `useState` 等の hooks を提供しています。
詳しい仕組みは~~正直わかりません~~省きますがざっくり説明すると、例えば `useState` の場合では React 内部にデータ領域が作られて、更新関数を用いることで内部のデータを更新するとともに、必要があれば再レンダリングを行うような流れになっています。

UIを構成するためのデータには様々なものがあり、大きく2つに分けることができると考えています。1つ目は 
この記事で主張したいこととしては、 **React の外部にあるデータを React で管理しようしない** ということです。

本記事で主張したい内容は「React 外のデータを Component や hooks 等で利用する際に `useState` や `useReducer` の利用を今すぐ中止し、全て `useSyncExternalStore` を用いよ」