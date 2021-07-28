---
title: "スマホでコード書く CI編"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: true
---

最初にCI/CDを敷こうと思います。空いた時間を使って開発をする際に開発が滞ることは避けたいです。テストを書いて、滞りのない開発を実現します。

## NuxtのCI/CD

GitHubで管理するのでCIにはGitHub Actionsを使います。Nuxt側はcreate-nuxt-appでスキャフォールディングした際に既に.github/workflows/ci.yamlが作られているので現状特に追加の設定は必要ありません。テストすべきロジックを切り出した際に追い追い書いて行くとしましょう。

LSPやALEによるリントエラーのリアルタイム表示は使わないこととします。パフォーマンスの問題もありますし、CLIで行えば十分です。vimを使っているので、:!npm run lintとすればエディタを立ち上げながらCLIでLinterが使えます。CIと同じLintをかけている安心感もあります。

```json
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "nuxt",
    "build": "nuxt build",
    "start": "nuxt start",
    "generate": "nuxt generate",
    "lint:prettier": "prettier --write --ignore-path .gitignore ./**/*.{ts,js,vue}",
    "lint:js": "eslint --fix --ext \".js,.ts,.vue\" --ignore-path .gitignore .",
    "lint": "npm run lint:prettier && npm run lint:js",
    "test": "jest"
  },
  "dependencies": {
    "@nuxtjs/axios": "^5.13.6",
    "core-js": "^3.15.1",
    "nuxt": "^2.15.7",
    "nuxt-property-decorator": "^2.9.1",
    "vuetify": "^2.5.5"
  },
  "devDependencies": {
    "@babel/eslint-parser": "^7.14.7",
    "@nuxt/types": "^2.15.7",
    "@nuxt/typescript-build": "^2.1.0",
    "@nuxtjs/eslint-config-typescript": "^6.0.1",
    "@nuxtjs/eslint-module": "^3.0.2",
    "@nuxtjs/vuetify": "^1.12.1",
    "@vue/test-utils": "^1.2.1",
    "babel-core": "7.0.0-bridge.0",
    "babel-jest": "^27.0.5",
    "eslint": "^7.29.0",
    "eslint-config-prettier": "^8.3.0",
    "eslint-plugin-nuxt": "^2.0.0",
    "eslint-plugin-vue": "^7.12.1",
    "jest": "^27.0.5",
    "prettier": "^2.3.2",
    "ts-jest": "^27.0.3",
    "vue-jest": "^3.0.4"
  }
}
```

フォーマッタとリンターを別々に叩くのが今風のようです。最初eslintのプラグインを入れようと考えていましたが、確認したところ推奨設定が変わっていました。

nuxt-property-decoratorが入ってなかったのでインストールしました。Vue3ではComposition APIが主流となりvue-property-decoratorはあまり採用されないようなのですが、Nuxt2では現役の書き方だと認識しています。仕事で使っているわけではないという性質上エクスペリメンタルなNuxt向けのComposition APIも視野には入りますが、同時の二つ以上の挑戦を避ける形としました。

layouts/default.vueをnuxt-property-decoratorで書き換えます。

```html
<template>
  <v-app dark>
    <v-navigation-drawer
      v-model="drawer"
      :mini-variant="miniVariant"
      :clipped="clipped"
      fixed
      app
    >
      <v-list>
        <v-list-item
          v-for="(item, i) in items"
          :key="i"
          :to="item.to"
          router
          exact
        >
          <v-list-item-action>
            <v-icon>{{ item.icon }}</v-icon>
          </v-list-item-action>
          <v-list-item-content>
            <v-list-item-title v-text="item.title" />
          </v-list-item-content>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-app-bar :clipped-left="clipped" fixed app>
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
      <v-btn icon @click.stop="miniVariant = !miniVariant">
        <v-icon>mdi-{{ `chevron-${miniVariant ? 'right' : 'left'}` }}</v-icon>
      </v-btn>
      <v-btn icon @click.stop="clipped = !clipped">
        <v-icon>mdi-application</v-icon>
      </v-btn>
      <v-btn icon @click.stop="fixed = !fixed">
        <v-icon>mdi-minus</v-icon>
      </v-btn>
      <v-toolbar-title v-text="title" />
      <v-spacer />
      <v-btn icon @click.stop="rightDrawer = !rightDrawer">
        <v-icon>mdi-menu</v-icon>
      </v-btn>
    </v-app-bar>
    <v-main>
      <v-container>
        <Nuxt />
      </v-container>
    </v-main>
    <v-navigation-drawer v-model="rightDrawer" :right="right" temporary fixed>
      <v-list>
        <v-list-item @click.native="right = !right">
          <v-list-item-action>
            <v-icon light> mdi-repeat </v-icon>
          </v-list-item-action>
          <v-list-item-title>Switch drawer (click me)</v-list-item-title>
        </v-list-item>
      </v-list>
    </v-navigation-drawer>
    <v-footer :absolute="!fixed" app>
      <span>&copy; {{ new Date().getFullYear() }}</span>
    </v-footer>
  </v-app>
</template>

<script lang="ts">
import { Component, Vue } from 'nuxt-property-decorator';

@Component
export default class DefaultLayout extends Vue {
  clipped: boolean = false;
  drawer: boolean = false;
  fixed: boolean = false;
  items: Array<any> = [
    {
      icon: 'mdi-apps',
      title: 'Welcome',
      to: '/',
    },
    {
      icon: 'mdi-chart-bubble',
      title: 'Inspire',
      to: '/inspire',
    },
  ];

  miniVariant: boolean = false;
  right: boolean = false;
  rightDrawer: boolean = false;
  title: string = 'Vuetify.js';
}
</script>
```

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

次回はgolangci-lintでlintを設定していきます。
