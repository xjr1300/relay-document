# 永続的なクエリ

<https://relay.dev/docs/guides/persisted-queries/>

- [永続的なクエリ](#永続的なクエリ)
  - [クライアントでの使用方法](#クライアントでの使用方法)
    - [persistConfigオプション](#persistconfigオプション)
      - [1. すべてのクエリとミューテーション操作テキストをmd5ハッシュに変換する](#1-すべてのクエリとミューテーション操作テキストをmd5ハッシュに変換する)
      - [2. 指定した`url`に`text`パラメーターを付けてHTTPのPOSTリクエストを送信する](#2-指定したurlにtextパラメーターを付けてhttpのpostリクエストを送信する)
    - [ローカルな永続的なクエリ](#ローカルな永続的なクエリ)
      - [得失評価](#得失評価)
    - [relayLocalPersisting.jsの実装例](#relaylocalpersistingjsの実装例)
    - [ネットワークレイヤの変更](#ネットワークレイヤの変更)
  - [サーバーで永続的なクエリを実行する](#サーバーで永続的なクエリを実行する)
    - [コンパイル時の配置](#コンパイル時の配置)
    - [ラインタイムの配置](#ラインタイムの配置)
    - [単純なサーバーの例](#単純なサーバーの例)
  - [persistConfigと--watchの使用](#persistconfigと--watchの使用)

Relayコンパイラーは永続的なクエリをサポートします。
これが役に立つ理由を次に示します。

- クライアントの操作テキストが単なるmd5ハッシュに成り、それは通常本物のクエリ文字列よりも短くなります。
  これはクライアントからサーバーへアップロードするバイト数を節約します。
- サーバーは、クライアントによって実行される操作を制限することで、セキュリティを改善するクエリの許可リストを作成できます。

## クライアントでの使用方法

### persistConfigオプション

`package.json`内のRelay設定セクションで、"persistConfig"を指定する必要があります。

```json
// package.json
{
  ...
  "scripts": {
    "relay": "relay-compiler",
    "relay-persisting": "node relayLocalPersisting.js"
  },
  "relay": {
    "src": "./src",
    "schema": "./schema.graphql",
    "persistConfig": {
      "url": "http://localhost:2999",
      "params": {}
    }
  }
  ...
}
```

設定内に`persistConfig`を指定する方法は次のとおりです。

#### 1. すべてのクエリとミューテーション操作テキストをmd5ハッシュに変換する

例えば`persistConfig`なしで、生成された`ConcreteRequest`は次のようになります。

```js
const node/*: ConcreteRequest*/ = (function() {
  ...
  return {
    "kind": "Request",
    "operationKind": "query",
    "name": "TodoItemRefetchQuery",
    "id": null, // idがnullであることに注意してください。
    "text": "query TodoItemRefetchQuery(\n  $itemID: ID!\n) {\n  node(id: $itemID) {\n    ...TodoItem_item_2FOrhs\n  }\n}\n\nfragment TodoItem_item_2FOrhs on Todo {\n    text\n    isComplete\n}\n",
    ...
  };
  })();
```

`persistConfig`を使用すると次のようになります。

```js
const node/*: ConcreteRequest*/ = (function() {
  ...
  return {
    "kind": "Request",
    "operationKind": "query",
    "name": "TodoItemRefetchQuery",
    "id": "3be4abb81fa595e25eb725b2c6a87508", // idがクエリテキストのmd5ハッシュになっていることに注意してください。
    "text": null, // textがnullになっていることに注意してください。
    ...
  };
  })();
```

#### 2. 指定した`url`に`text`パラメーターを付けてHTTPのPOSTリクエストを送信する

また、`param`オプションを介してリクエストボディパラメーターを追加することもできます。

```json
// package.json
{
  ...
  "scripts": {
    "relay": "relay-compiler"
  },
  "relay": {
    "src": "./src",
    "schema": "./schema.graphql",
    "persistConfig": {
      "url": "http://localhost:2999",
      "params": {}
    }
  }
  ...
}
```

### ローカルな永続的なクエリ

次の設定を使用して、`operation_id => full operation text`のマップを含むローカルなJSONファイルを生成でします。

```json
// package.json
{
  ...
  "scripts": {
    "relay": "relay-compiler"
  },
  "relay": {
    "src": "./src",
    "schema": "./schema.graphql",
    "persistConfig": {
      "file": "./persisted_queries.json",
      "algorithm": "MD5" // this can be one of MD5, SHA256, SHA1
    }
  }
  ...
}
```

理想的には、このファイルを取り上げ、それをデプロイ時にサーバーに送ると、サーバーはサーバーが受け取る可能性のあるすべてのクエリを理解します。
もし、それを望まないの場合、[自動永続的クエリ応答確認](https://www.apollographql.com/docs/apollo-server/performance/apq/)を実装する必要があります。

#### 得失評価

- ✅ もしサーバーの永続的なクエリデータストアが除かれた場合、クライアントのリクエストを介して自動的に復元できます。
- ❌ キャッシュミスがあるとき、サーバーへの追加のラウンドトリップを支払うでしょう。
- ❌ バンドルサイズの増加を伴う、`persisted_queries.json`ファイルをブラウザに送らなければなりません。

### relayLocalPersisting.jsの実装例

ここに、`queryMap.json`ファイルにクエリテキストを保存する単純な永続的サーバーの例を示します。

```js
const http = require('http');
const crypto = require('crypto');
const fs = require('fs');

function md5(input) {
  return crypto.createHash('md5').update(input).digest('hex');
}

class QueryMap {
  constructor(fileMapName) {
    this._fileMapName = fileMapName;
    this._queryMap = new Map(JSON.parse(fs.readFileSync(this._fileMapName)));
  }

  _flush() {
    const data = JSON.stringify(Array.from(this._queryMap.entries()));
    fs.writeFileSync(this._fileMapName, data);
  }

  saveQuery(text) {
    const id = md5(text);
    this._queryMap.set(id, text);
    this._flush();
    return id;
  }
}

const queryMap = new QueryMap('./queryMap.json');

async function requestListener(req, res) {
  if (req.method === 'POST') {
    const buffers = [];
    for await (const chunk of req) {
      buffers.push(chunk);
    }
    const data = Buffer.concat(buffers).toString();
    res.writeHead(200, {
      'Content-Type': 'application/json'
    });
    try {
      if (req.headers['content-type'] !== 'application/x-www-form-urlencoded') {
        throw new Error(
          'Only "application/x-www-form-urlencoded" requests are supported.'
        );
      }
      const text = new URLSearchParams(data).get('text');
      if (text == null) {
        throw new Error('Expected to have `text` parameter in the POST.');
      }
      const id = queryMap.saveQuery(text);
      res.end(JSON.stringify({"id": id}));
    } catch (e) {
      console.error(e);
      res.writeHead(400);
      res.end(`Unable to save query: ${e}.`);
    }
  } else {
    res.writeHead(400);
    res.end("Request is not supported.")
  }
}

const PORT = 2999;
const server = http.createServer(requestListener);
server.listen(PORT);

console.log(`Relay persisting server listening on ${PORT} port.`);
```

上記例は、`./queryMap.json`への完全なクエリマップファイルを記述しています。
これを使用するために、`package.json`を更新する必要があります。

```json
// package.json
{
  ...
  "scripts": {
    "persist-server": "node ./relayLocalPersisting.js",
    "relay": "relay-compiler"
  }
  ...
}
```

### ネットワークレイヤの変更

クエリパラメーターの代わりに、例えば`doc_id`など、POSTボディ内にIDをパラメーターを渡すために、ネットワークレイヤの取得実装を修正する必要があります。

```js
function fetchQuery(operation, variables) {
  return fetch('/graphql', {
    method: 'POST',
    headers: {
      'content-type': 'application/json'
    },
    body: JSON.stringify({
      doc_id: operation.id, // md5ハッシュをサーバーに渡していることに注意してください。
      // query: operation.text, // テキストはnullであるため、これは現在痕跡だけです。
      variables,
    }),
  }).then(response => {
    return response.json();
  });
}
```

## サーバーで永続的なクエリを実行する

クエリテキストの代わりに送信された永続的なクエリを送信するクライアントリクエストを実行するために、それぞれのIDに対応するクエリテキストを探すことができる必要があります。
通常、これは、`queryMap.json`JSONファイルの出力をデータベースまたは他のストレージ装置に保存して、クライアントによって指定されたIDに対応するテキストを取得することを含みます。

さらに、`relayLocalPersisting.js`の実装は、データベースまたは他のストレージにクエリを保存します。

クライアントとサーバーのコードが1つのプロジェクト内にあるユニバーサルアプリケーションの場合、クライアントとサーバーの両方にアクセス可能な一般的な場所にあるクエリマップファイルを配置できるため、これは問題になりません。

### コンパイル時の配置

クライアントとサーバーのプロジェクトが分離されているアプリケーションの場合、1つの選択肢はサーバーによってアクセス可能な場所に、コンパイル時にクエリマップファイルを配置する追加的なnpmのrunスクリプトを持つことです。

```json
// package.json
{
  ...
  "scripts": {
    "push-queries": "node ./pushQueries.js",
    "persist-server": "node ./relayLocalPersisting.js",
    "relay": "relay-compiler && npm run push-queries"
  }
  ...
}
```

`./pushQueries.js`ですることができるいくつかのことがあります。

- サーバーのリポジトリに`git push`します。
- データベースにクエリマップを保存します。

### ラインタイムの配置

2番目のより複雑な選択肢は、最初はサーバーがクエリのIDを理解することなしで、ランタイムでサーバーにクエリマップファイルを配置することです。
クライアントは、クエリマップを持っていないサーバーに、楽観的にクエリIDを送信します。
そして、次にサーバーはクライアントから完全なクエリテキストをリクエストされるため、サーバーは後続するリクエストのためにクエリマップをキャッシュできます。
これは、クエリマップを交換するために相互作用するクライアントとサーバーを要求するより複雑な方法です。

### 単純なサーバーの例

一度サーバーがクエリマップにアクセスすれば、マッピングを実行できます。
この解決方法は、使用しているサーバーとデータベース技術に依存して異なるため、最も一般的で基本的な例を説明します。

`express-graphql`とクエリマップへのアクセスできる場合、それを直接インポートして、[express-graphql-persisted-queries](https://github.com/kyarik/express-graphql-persisted-queries)の`persistedQueries`ミドルウェアを使用してマッチングを実行できます。

```js
import express from 'express';
import {graphqlHTTP} from 'express-graphql';
import {persistedQueries} from 'express-graphql-persisted-queries';
import queryMap from './path/to/queryMap.json';

const app = express();

app.use(
  '/graphql',
  persistedQueries({
    queryMap,
    queryIdKey: 'doc_id',
  }),
  graphqlHTTP({schema}),
);
```

## persistConfigと--watchの使用

`persistConfig`と`--watch`オプションを同時に使用することで、継続的にクエリマップファイルを生成することが可能です。
これはユニバーサルアプリケーションでのみ意味をなすため、例えば、クライアントとサーバーのコードが1つのプロジェクト内にあり、開発中にlocalhostで両方を一緒に実行します。
さらに、`queryMap.json`への変更を取り上げるために、サーバーのホットリロードを準備する必要があります。
これを準備する方法の詳細は、このドキュメントの範囲外です。
