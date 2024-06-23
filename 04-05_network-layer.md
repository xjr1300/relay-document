# ネットワークレイヤ

<https://relay.dev/docs/guides/network-layer/>

- [ネットワークレイヤ](#ネットワークレイヤ)
  - [キャッシング](#キャッシング)

GraphQLサーバーにアクセスする方法を知るために、Relayは、Relay環境のインスタンスを作成するときに、`INetwork`インターフェイスを実装したオブジェクトを提供することを開発者に要求します。
その環境は、クエリ、ミューテーション、そしてサーバーがサポートしている場合はサブスクリプションを実行するためにこのネットワークレイヤを使用します。
これにより、開発者はアプリケーションに最適なHTTP、WebSocketなどと認証を使用して、それぞれのアプリケーションのネットワーク構成の詳細から環境を切り離すことができます。

現在、ネットワークレイヤを作成する最も簡単な方法は、`relay-runtime`パッケージのヘルパーを使用することです。

```js
import {
  Environment,
  Network,
  RecordSource,
  Store,
} from 'relay-runtime';

// クエリ、ミューテーションなどの操作の結果を取得して、Promiseとしてその結果を返す
// 関数を定義します。
function fetchQuery(
  operation,
  variables,
  cacheConfig,
  uploadables,
) {
  return fetch('/graphql', {
    method: 'POST',
    headers: {
      // ここで認証または他のヘッダーを追加します。
      'content-type': 'application/json'
    },
    body: JSON.stringify({
      query: operation.text, // 入力のGraphQLテキスト
      variables,
    }),
  }).then(response => {
    return response.json();
  });
}

// fetch関数からネットワークレイヤを作成する。
const network = Network.create(fetchQuery);
const store = new Store(new RecordSource())

const environment = new Environment({
  network,
  store
  // ... 他のオプション
});

export default environment;
```

これは、始めるための基本的な例であることに注意してください。
この例は、`cacheConfig.force`がfalseのときに有効するリクエストとレスポンスのキャッシングや、`uploadable`パラメーターを使用したミューテーション用のデータフォームのアップロードのような追加的な機能で拡張されます。

## キャッシング

Relayストアは、現在保持されているクエリからのデータをキャッシュします。
ガイドツアーの[キャッシュされたデータの再利用](https://relay.dev/docs/guided-tour/reusing-cached-data/)のセクションを参照してください。
