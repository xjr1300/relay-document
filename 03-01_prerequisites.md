# 前提条件

<https://relay.dev/docs/getting-started/prerequisites/>

- [前提条件](#前提条件)
  - [JavaScript](#javascript)
  - [React](#react)
  - [GraphQL](#graphql)
    - [GraphQLスキーマ](#graphqlスキーマ)
    - [GraphQLサーバー](#graphqlサーバー)

Relayを開始する前に、次のインフラストラクチャーがすでに準備されており、下のトピックについてある程度の知識があることを想定していることを心に留めておいてください。

## JavaScript

RelayはJavaScriptで構築されたフレームワークであるため、JavaScript言語に慣れていることを想定しています。

## React

Relayは、Reactアプリケーション用に主要なバインディングをサポートしたデータ管理フレームワークであるため、すでに[React](https://reactjs.org/)に慣れていることを想定しています。

## GraphQL

[GraphQL](http://graphql.org/learn/)の基本的な理解を持っていることを想定しています。
Relayの使用を開始するために、次も必要になります。

### GraphQLスキーマ

アプリケーションに必要な任意のデータを取得する方法を理解するリゾルブメソッドの集合と関連するデータモデルの記述もまた必要です。

GraphQLは、幅広い範囲のデータアクセスパターンをサポートするために設計されています。
アプリケーションのデータ構造を理解するために、Relayはスキーマを定義するときに、特定の慣習に従うことを要求します。
これらは、[GraphQLサーバー仕様](https://relay.dev/docs/guides/graphql-server-specification/)に文書化されています。

- [npm](https://www.npmjs.com/package/graphql)の[graphql-js](https://github.com/graphql/graphql-js)

  JavaScriptを使用したGraphQLスキーマを構築するための汎用ツールです。

- [npm](https://www.npmjs.com/package/graphql-relay)の[graphql-relay-js](https://github.com/graphql/graphql-relay-js)

  データ間のコネクション、Relayと円滑に統合する方法で、ミューテーションを定義するためのJavaScriptのヘルパーです。

### GraphQLサーバー

スキーマをロードして、GraphQLを話すことを教えられたに2のサーバーです。
[例](https://github.com/relayjs/relay-examples)はExpressを使用しています。

- [npm](https://www.npmjs.com/package/express-graphql)の[express-graphql](https://github.com/graphql/express-graphql)
