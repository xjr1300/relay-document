# インストール

<https://relay.dev/docs/getting-started/installation-and-setup/>

- [インストール](#インストール)
  - [手動インストール](#手動インストール)
  - [コンパイラーのセットアップ](#コンパイラーのセットアップ)
  - [コンパイラーの設定](#コンパイラーの設定)
  - [babel-plugin-relayのセットアップ](#babel-plugin-relayのセットアップ)
  - [コンパイラーの実行](#コンパイラーの実行)
  - [JavaScript環境の要求事項](#javascript環境の要求事項)

多くの状況で、Relayをインストールする最も簡単な方法は、Tobias Tenglerによって記述された`create-relay-app`パッケージを使用することです。
名前の暗示とは対照的に、このパッケージは既存のアプリにRelayを*インストール*します。

現在、それはNext、Vite、そしてCreate React Appで構築されたアプリをサポートしてます。
それらのプラットフォームの1つ出ない場合、また何らかの理由で機能しない場合、下の手動ステップに進んでください。

それを実行するために、きれいな作業ディレクトリであることを確認して、次を実行してください。

```sh
npm create @tobiastengler/relay-app
```

好みに応じて、`npm`の代わりに`yarn`、`pnpm`を使用してください。

それが完了したとき、それは従うべき"Next Steps"をいくつか表示します。

このスクリプトの詳細は、[GitHubリポジトリ](https://github.com/tobias-tengler/create-relay-app)で見つけることができます。

---

## 手動インストール

`yarn`または`npm`を使用してReactとRelayをインストールしてください。

```shell
yarn add react react-dom react-relay
```

## コンパイラーのセットアップ

Relayの事前コンパイルは、[Relayコンパイラー](https://relay.dev/docs/guides/compiler/)を要求して、それは`yarn`または`npm`を介してインストールできます。

```shell
yarn add --dev relay-compiler
```

これは、node_modulesフォルダ内に`relay-compiler`実行スクリプトをインストールします。
`package.json`ファイルにスクリプトを追加して、`yarn/npm`スクリプトからこれを実行することが推奨されます。

```json
// package.json
{
  "scripts": {
    "relay": "relay-compiler"
  }
}
```

## コンパイラーの設定

設定ファイルを作成してください。

```js
// relay.config.js
module.exports = {
  // ...
  // `relay-compiler`コマンドラインツールと`babel-plugin-relay`によって受け入れられる設定オプション
  src: "./src",
  language: "javascript", // "javascript" | "typescript" | "flow"
  schema: "./data/schema.graphql",
  excludes: ["**/node_modules/**", "**/__mocks__/**", "**/__generated__/**"],
}
```

また、この設定は`package.json`の`"relay"`セクション内に記述することもできます。
詳細と設定オプションは[Relayコンパイラー設定](https://github.com/facebook/relay/tree/main/packages/relay-compiler)を確認してください。

## babel-plugin-relayのセットアップ

Relayは、GraphQLをランタイムな人工物に変換するために、Babelプラグインを要求します。

```sh
yarn add --dev babel-plugin-relay graphql
```

`.babelrc`ファイルのプラグインのリストに`"relay"`を追加してください。

```text
# .babelrc
{
  "plugins": ["relay"]
}
```

`"relay"`プラグインは、他のプラグインの前に実行されるか、`graphql`テンプレートリテラルが正確に変換されることを確認することを事前に調整されるべきであることに注意してください。
このトピックにいて[Babelのドキュメント](https://babeljs.io/docs/plugins/#pluginpreset-ordering)を参照してください。

`babel-plugin-relay`を使用する代わりに、[babel-plugin-macros](https://github.com/kentcdodds/babel-plugin-macros)でRelayを使用できます。
`babel-plugin-macros`をインストールした後、Babelの設定に次を追加してください。

```javascript
const graphql = require('babel-plugin-relay/macro');
```

## コンパイラーの実行

アプリケーションのファイルを編集した後、新しいコンパイルされた人工物を生成するために、`relay`スクリプトを実行してください。

```sh
yarn run relay
```

代わりに、ソースコードを確認して、自動てコンパイルされた人工物を再生成するために、`--watch`オプションを渡すことができます。
ただし、注意事項として、これは[watchman](https://facebook.github.io/watchman)のインストールされていることを要求します。

```sh
yarn run relay --watch
```

詳細は、[Relayコンパイラーのドキュメント](https://relay.dev/docs/guides/compiler/)を参照してください。

## JavaScript環境の要求事項

NPMで配信されているRelayパッケージは、可能な限り多くのブラウザ環境をサポートするために、広くサポートされたJavaScriptのES5バージョンを使用しています。

しかし、Relayは、定義されるために、`Map`、`Set`、`Promise`、`Object.assign`など、現代のJavaScriptのグローバルな型を予期しています。
これらをネイティブに提供していないかもしれない古いブラウザやデバイスをサポートする場合、バンドルされたアプリケーションに、[core-js](https://relay.dev/docs/guides/compiler/)または[babel/polyfill](https://babeljs.io/docs/usage/polyfill/)のようなグローバルなポリフィルを含めることを検討してください。

古いブラウザをサポートするために`core-js`を使用したRelayのポリフィルされた環境は、次のようになります。

```js
require('core-js/es6/map');
require('core-js/es6/set');
require('core-js/es6/promise');
require('core-js/es6/object');

require('./myRelayApplication');
```
