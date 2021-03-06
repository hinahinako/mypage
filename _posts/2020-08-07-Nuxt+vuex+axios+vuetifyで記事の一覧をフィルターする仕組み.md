---
layout: article
title: Nuxt+vuex+axios+vuetifyで記事の一覧をフィルターする仕組み
tags: Nuxt.js
aside:
  toc: true
---


# はじめに

##この記事について
axiosで外部データを取得し、vuexで管理する方法を知りました。
その方法を忘れてしまった未来の自分のために、内容をこの記事にまとめておきます。

環境:
- Nuxt.js (2.14.1)
- vuex (^5.12.0)
- axios (^1.11.2)
- Vuetify (^2)


## やること
![image](https://user-images.githubusercontent.com/44778704/89621529-74a32b80-d8cc-11ea-8bef-c8f155c31c64.png)

- axiosで外部APIから記事一覧を取得する
- 記事一覧はstore側で管理し、page側で表示する
- userIdで記事をフィルターできるようにする


## 外部から取得するデータ

以下のデータを使います。
[JSONPlaceholder](https://jsonplaceholder.typicode.com/posts)

中身は以下のような形式になっています。

```json
[
  {
    "userId": 1,
    "id": 1,
    "title": "title text",
    "body": "body text"
  }
]
```


# 1.プロジェクト作成

nuxt-cliでプロジェクトを作成します。

```
yarn create nuxt-app プロジェクト名
```

選択は以下の通りです。
![image](https://user-images.githubusercontent.com/44778704/89613608-9a750400-d8bd-11ea-9d60-93de6a632ff7.png)

ここでVuetifyの選択を忘れていたので後から追加しました。。

```
yarn add @nuxtjs/vuetify
```

```
// nuxt.config.js

  devModules: ['@nuxtjs/vuetify'],
  vuetify: {
      theme: {
        dark: false
      }
    },
```

#### 補足

以下の警告が出ていました。
```WARN devModules has been renamed to buildModules and will be removed in Nuxt 3. 15:11:12```

devModulesはNuxt3以降で消える予定らしいです。
今回は気にせず続行します！

# 結論

以下2つのファイルをつくれば上記の実装が可能です。

## `store/posts.js`
store配下に`posts.js`を作成します。

```js

// store/posts.js

export const state = () => ({
  list: [],
  viewList: [],
  length: 0
});

export const mutations = {
  setList(state, list) {
    state.list = list;
  },
  setViewList(state, viewList) {
    state.viewList = viewList;
  },
  setLength(state, length) {
    state.length = length;
  }
};

export const actions = {
  // APIから記事を取得する
  async getList({ commit }, categories) {
    const list = await this.$axios.$get(
      "https://jsonplaceholder.typicode.com/posts"
    );

    // 記事数が100件だとサンプルとして紹介しにくいので3分の1に絞りこむ
    const oddIdList = list.filter(article => article.id % 3 === 0);

    commit("setList", oddIdList);
    commit("setViewList", oddIdList);
    commit("setLength", oddIdList.length);
  },

  // 選択されたカテゴリに一致する記事を絞り込む
  filterUser({ commit, state }, selectedId) {
    const viewList = state.list.filter(
      article => article.userId === selectedId
    );
    commit("setViewList", viewList);
    commit("setLength", viewList.length);
  }
};

```

## `pages/posts.vue`
pages配下に`posts.vue`を作成します。

```html
<!-- pages/posts.vue -->

<template>
  <v-main>
    <v-container>
      <v-row justify="center">
        <v-btn @click="filterUser(1)">user 1</v-btn>
        <v-btn @click="filterUser(2)">user 2</v-btn>
        <v-btn @click="filterUser(3)">user 3</v-btn>
        <v-col>
          <p>記事の数: {{length}}</p>
        </v-col>
      </v-row>
      <v-row>
        <v-list>
          <v-list-item v-for="(article, i) in viewList" :key="i">
            <v-list-item-content>
              <p>userId:{{article.userId}}</p>
              <v-list-item-title>{{ article.title }}</v-list-item-title>
              <v-list-item-subtitle>{{ article.body }}</v-list-item-subtitle>
            </v-list-item-content>
          </v-list-item>
        </v-list>
      </v-row>
    </v-container>
  </v-main>
</template>

<script>
import { mapActions, mapState } from "vuex";

export default {
  name: "news",
  data() {
    return {};
  },
  methods: {
    ...mapActions({
      getList: "posts/getList",
      filterUser: "posts/filterUser",
    }),
  },
  mounted: function () {
   this.getList(this.categories);
  },
  computed: {
    ...mapState({
      viewList: (state) => state.posts.viewList,
      length: (state) => state.posts.length,
    }),
  },
};
</script>

```

# `store/posts.jsの解説`

上から順に解説します。

## state

StateはコンポーネントでいうところのDataみたいなものです。

```js
export const state = () => ({
  list: [],
  viewList: [],
  length: 0
});
```

ここでは3つの変数を定義しています。

- `list`・・・APIから取得した記事をそのまま格納する変数
- `viewList`・・・フィルターした後のlistを入れる変数
- `length`・・・viewListの記事数を入れる変数

## Mutations

Mutationsは、コンポーネントでいうとMethodsのようなものです。
Stateの状態を更新する際はこのMutationを使います。

```js
export const mutations = {
  setList(state, list) {
    state.list = list;
  },
  setViewList(state, viewList) {
    state.viewList = viewList;
  },
  setLength(state, length) {
    state.length = length;
  }
};
```

参考:
- [Vuexのミューテーションの使い方。状態を更新しよう｜Vuex](https://dev83.com/vue-vuex04/)

今回はActionsで処理した結果を受け取り、Stateに再代入するだけの関数をそれぞれ用意しています。
いわゆるsetterです。

- `setList`
- `setViewList`
- `setLength`

## Actions

Actionsは非同期処理を書くことができます。
今回はactionsで色々処理し、Mutationを通してStateを更新するという流れで書いています。

(より良いActionsとMutationsの違いや使い分けについてはまだ勉強中です。。。)

### `getList()`

`getList()`は外部から記事一覧を取得し、一覧をすべて表示するための関数です。

```js
export const actions = {
  // APIから記事を取得する
  async getList({ commit }, categories) {
    const list = await this.$axios.$get(
      "https://jsonplaceholder.typicode.com/posts"
    );
```
ここではasync/awaitを使って外部の記事を取得しています。
Actionsでは非同期処理を記述できるので、axiosで外部APIを叩くような処理はここに書きます。


```js

// 記事数が100件だとサンプルとして紹介しにくいので3分の1に絞りこむ
    const oddIdList = list.filter(article => article.id % 3 === 0);

```

コメントにある通りですが・・・
この記事は100件もあってサンプルとして使いにくかったので、適当に34件に絞り込みました。

```js
    commit("setList", oddIdList);
    commit("setViewList", oddIdList);
    commit("setLength", oddIdList.length);
  },
```

`commit`で処理結果を引数にMutationsを実行し、Stateの状態を更新しています。

### `filterUser()`
filterUser()はユーザIDに該当する記事だけを表示するための関数です。

```js
  // 選択されたカテゴリに一致する記事を絞り込む
  filterUser({ commit, state }, selectedId) {
    const viewList = state.list.filter(
      article => article.userId === selectedId
    );
```

listに格納されているデータの中から、userIdが引数と一致するものだけを抽出しています。

```js
    commit("setViewList", viewList);
    commit("setLength", viewList.length);
  }
```

`commit`で処理結果を引数にmutationsを実行し、stateの状態を更新しています。


### メモ dispatchについて

ActionsからActionsを呼びたい時は`dispatch`を使います。
例えば`getList`から`filterUser()`を呼ぶ場合は以下のように書きます。

```js
async getList({ commit,dispatch }, categories) {
    const list = await this.$axios.$get(
      "https://jsonplaceholder.typicode.com/posts"
    );
    commit("setViewList", viewList);
    commit("setLength", viewList.length);
    dispatch("filterUser", 1)
})
```

# `pages/posts.vue`

## `<template>`

### ボタン

![image](https://user-images.githubusercontent.com/44778704/89707499-8c041680-d9a9-11ea-9ba1-37a7be5bc088.png)

```html
<v-btn @click="filterUser(1)">user 1</v-btn>
<v-btn @click="filterUser(2)">user 2</v-btn>
<v-btn @click="filterUser(3)">user 3</v-btn>
```

3つのボタンを用意しています。
`user 1`を押すと、`filterUser()`を実行し、userIdが`1`の記事を絞り込みます。


### 記事数

![image](https://user-images.githubusercontent.com/44778704/89707541-fc129c80-d9a9-11ea-891c-b1f22fbd602e.png)

```html
<v-col>
  <p>記事の数: {{length}}</p>
</v-col>
```
記事数を表示しています。

### 記事一覧

![image](https://user-images.githubusercontent.com/44778704/89707570-46941900-d9aa-11ea-8e29-62ab51577c49.png)

```html
<v-list>
  <v-list-item v-for="(article, i) in viewList" :key="i">
    <v-list-item-content>
      <p>userId:{{article.userId}}</p>
      <v-list-item-title>{{ article.title }}</v-list-item-title>
      <v-list-item-subtitle>{{ article.body }}</v-list-item-subtitle>
    </v-list-item-content>
  </v-list-item>
</v-list>
```
ここではVuetifyの`v-lis`を使って記事一覧を表示しています。
`v-for="(article, i) in viewList" :key="i"`のところでviewList内の記事を順次取り出し、マスタッシュ `{{}}`を使って記事の内容を表示しています。

## `<script>`

```js
import { mapActions, mapState } from "vuex";
```
mapActionsとmapStateをインポートして使用できるようにしています。
これらはヘルパーというらしいです。

ヘルパーの種類:

- 1.状態を呼び出す
  - mapState
  - mapGetters

- 2.状態を変化させる
  - mapMutations
  - mapActions

参考:
[【Vuex】mapState, mapGetters, mapMutations, mapActionsの最低限の使い方まとめ](https://qiita.com/terufumi1122/items/6f9470c8d416a4af7502)


### Methods

```js
methods: {
    ...mapActions({
      getList: "posts/getList",
      filterUser: "posts/filterUser",
    }),
  },
```
mapActionsを使ってstoreのactionsをmethodsから呼び出せるように定義しています。

- `getList`・・・actionsの`getList()`を呼ぶ
- `filterUser`・・・actionsの`filterUser()`を呼ぶ

### Mounted

```js
mounted: function () {
  this.getList(this.categories);
  },
```

初期表示の際は全ての記事を表示するため、getListを実行しています。


#### 補足:

別のプロジェクトでこれを実行した際、記事が表示されない事象があった。
その際はasync/awaitで呼び出したら記事が表示された。

```js
mounted: async function () {
    await this.getList(this.categories);
  },
```
原因不明・・
まあこういう書き方もできるのか、くらいに覚えておこうと思いました。

### Computed

```js
computed: {
    ...mapState({
      viewList: (state) => state.posts.viewList,
      length: (state) => state.posts.length,
    }),
  },
```

Stateの変数をこちら側で使うために定義しています。
変数名はStateと同じでなくてもOKみたいです。

# まとめ

Methods→Actions→Mutationsの順に処理を実行してStateの値をいじるというのがVuexの基本です。多分。
別プロジェクトではおんなじ要領でページネーションも盛り込んでました。
なので、この記事がわかれば、基本的な操作はだいたいできるんじゃないかな、と思います。



# 参考
[ともすたチャンネル -Nuxt.js入門 #06：Vuexのストアでデータを保管しよう-](https://www.youtube.com/watch?v=h7HIVssdWTQ&list=PLh6V6_7fbbo8koq2fzoz8lN4gqtPC3Yg4&index=6&t=3s)※Youtube
