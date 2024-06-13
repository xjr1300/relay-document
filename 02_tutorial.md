# チュートリアル

## チュートリアルの紹介

<https://relay.dev/docs/tutorial/intro/>

このチュートリアルは、Relayの最も重要で、よく使用する機能で開始されます。
これを行うために、ニュースフィードを表示する単純なアプリを構築します。

- クエリを使用してデータを取得する方法
- クエリをフラブ面とに分割して、コンポーネントを自己完結させる方法
- コネクションでデータをページ分けする方法
- ミューテーションと更新でサーバーにあるデータを更新する方法

このチュートリアルは、Reactにかなり精通していることを想定しています。
もし、まだReactを初めて使用する場合は、[Reactのチュートリアル](https://reactjs.org/tutorial/)に進み、コンポーネントの作成、プロパティの私、そして`useSate`のような基本的なフックに慣れるまでReactを練習することを推奨します。
チュートリアルは、Webをベースとしていますが、RelayはReact Nativeでも素晴らしく機能します。

このチュートリアルは、TypeScriptで構築されいるため、[TypeScriptのとても基本的な知識](https://www.typescriptlang.org/docs/)はとても役に立ちます。
型の宣言やインポート、そして関数の注釈を超える何かを知る必要はありません。
また、Relayは、Flow型システムまたは他の型システムで使用することもできます。

- **ℹ INFO**
  **重要**: このチュートリアルは、順番に進むことを意図しているため、演習はそれぞれで構築されます。
  アプリ例に増分的に変更をしていくため、前の方のセクションを完了していない場合、後の方のセクションは意味がなくなります。

開始するため、次のコマンドを実行してください。

```sh
git clone https://github.com/relayjs/relay-examples.git
cd relay-examples/newsfeed
npm install
npm run dev
```

これは、開始するテンプレートプロジェクトをダウンロードして、サーバーを起動します。
もし、それらが機能しないのであれば、[Gitのインストール](https://github.com/git-guides/install-git)、または[npmのインストール](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)が必要かもしれません。

`npm run dev`を実行すると、いくつかのプロセスが開始されます。

- フロントエンドのコードを提供する、Webpackに基づくHTTPサーバー
- フロントエンドが、情報を取得するためにクエリする、基本的なGraphQLサーバー
- アプリ内でGraphQLを処理して、ランタイムでRelayが使用する追加的なファイルと同様に、入力とクエリの結果を表現するTypeScriptの型を生成するRelayコンパイラ

ターミナルの出力において、これら3つのプロセスのログは、黄色で`[webpack]`、緑色で`[relay]`、そして青色で`[relay]`のタグでマークされています。
`[relay]`でマークされたエラーに注意してください。
これらは、GraphQLに間違いがある場合に役立ちます。

`[relay]`プロセスが示す: `[relay] thread 'main' panicked at 'Cannot run relay in watch mode if ``watchman`` is not available (or explicitly disabled).'`のエラーに遭遇した場合、これはシステムに`watchman`がインストールされていないか利用できないことを意味します。
これを解決するために、[分離してwatchmanをインストール](https://facebook.github.io/watchman/docs/install)する必要があるかもしれません。
`watchman`をインストール後、再度`npm run dev`の実行を試してください。

現在、これらのプロセスは起動しています。
ブラウザで、<http://localhost:3000>を開くことができるはずです。

![page-that-start-app-at-first](https://relay.dev/assets/images/intro-screenshot-placeholder-b3306c44c795c65cf8ce0d2552e4d365.png)

Reactでレンダリングされた単一のニュースフィードストーリを法事するWebページから開始しますが、そのストールのデータは、Reactコンポーネントにハードコードされた単にプレイスホルダデータです。
このチュートリアルの残りに置いて、サーバーからデータを取得してデータを持ち、複数のストーリーをページ分けして、コメントやリンクを設定することによりデータを更新することによって、アプリを機能的にします。

アプリ例を彩るファイルは次の方法でレイアウトされます。

- `src/components` - 変更して作業するフロントエンドアプリのコンポーネントです。重要なコンポーネントのいくつかは次のとおりです。
  - `App.tsx` - 最上位コンポーネント
  - `Newsfeed.tsx` - ニュースフィードストーリを取得して、ストーリのスクローリングリストを表示します。
    チュートリアルの最初では、このコンポーネントはハードコードされたプレイスホルダを使用します。
    それをGraphQLとRelayでデータを取得するように修正します。
  - `Story.tsx` - 単一のニュースフィードストーリを表示します。
- `src/server` - データ例を提供するとても基本的なGraphQLサーバー
  - `server/schema.graphql` - GraphQLスキーマで、GraphQLを介してサーバーからクエリされる情報を記述します。

最後に、VSCodeを使用しているときに自動補完、エラー、そして他のヘルプを表示するために[Relay VSCode拡張機能](https://marketplace.visualstudio.com/items?itemName=meta.relay)をインストールしたいかもしれません。

次のセクションに進んで、GraphQLとRelayの学習を始めてください。
