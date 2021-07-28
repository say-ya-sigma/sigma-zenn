---
title: "スマホでコード書く CI編"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

最初にCI/CDを敷こうと思います。空いた時間を使って開発をする際に開発が滞ることは避けたいです。テストを書いて、滞りのない開発を実現します。

## NuxtのCI/CD

GitHubで管理するのでCIにはGitHub Actionsを使います。Nuxt側はcreate-nuxt-appでスキャフォールディングした際に既に.github/workflows/ci.yamlが作られているので現状特に追加の設定は必要ありません。テストすべきロジックを切り出した際に追い追い書いて行くとしましょう。

LSPやALEによるリントエラーのリアルタイム表示は使わないこととします。パフォーマンスの問題もありますし、CLIで行えば十分です。vimを使っているので、:!npm run lintとすればエディタを立ち上げながらCLIでLinterが使えます。CIと同じLintをかけている安心感もあります。

フォーマッタとリンターを別々に叩くのが今風のようです。最初eslintのプラグインを入れようと考えていましたが、確認したところ推奨設定が変わっていました。

nuxt-property-decoratorが入っていませんでした。Composition APIが来たVue3ではvue-property-decoratorはあまり採用されないようなのですが、Nuxtでは現役の書き方だと認識しているのでnuxt-property-decoratorを採用します。実験的プロダクトという性質上Nuxt向けのComposition APIも視野には入りますが、同時の二つ以上の挑戦を避ける形となりました。

layouts/default.vueを書き換えます。

## erudaを導入

書き換える際に、モバイルブラウザでチェックしているためconsoleにエラーが出ないことに困りました。erudaを入れて対処します。Vue devtoolsが使えないため、こればかりはPCに劣りそうです。また、インスペクタが使えません。

あえて利点を挙げるとするなら、モバイルファーストの時代で開発時の最初の検証環境がスマホだという点が挙げられるかもしれません。スマホ実機でのみ起こる不具合にはいち早く気がつけることでしょう。

### console

![](https://storage.googleapis.com/zenn-user-upload/1e46d7e8fd6c9a775367fec7.png)

### network

![](https://storage.googleapis.com/zenn-user-upload/b71611b8fab682b5a288c311.png)

### sources

![](https://storage.googleapis.com/zenn-user-upload/41f6f77fd9ac92844154d24c.png)

## golangci-lint

golangci-lintでlintを設定していきます。
