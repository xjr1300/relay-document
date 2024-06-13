# クエリの基本

<https://relay.dev/docs/tutorial/queries-1/>

- [クエリの基本](#クエリの基本)
  - [Deep Dive: データロードのためのSuspense](#deep-dive-データロードのためのsuspense)
  - [Deep Dive: クエリは静的](#deep-dive-クエリは静的)
  - [Relayと型システム](#relayと型システム)
  - [まとめ](#まとめ)

このセクションでは次を説明します。

- ハードコードされたプレイスホヅラデータを法事するReactコンポーネントを取り上げ、それがGraphQLクエリを使用してそのデータを取得するために修正します。
- 型安全性を保証するために、RelayがGraphQLから生成したTypeScriptの型を使用する方法を学びます。

---

Relayを使用して、GraphQLクエリを使用してデータを取得します。
クエリは、取得するためにアプリのGraphQLグラフの一部を記述して、木構造の特定のデータの集合を取得するために、任意のルートノードから開始して、ノードとノードの間を旅します。

![query and graph](https://relay.dev/assets/images/query-upon-graph-2209e828b9ce0ddc492555bb7a0a5a3c.png)

ただちに、アプリ例は任意のデータを取得せず、Reactコンポーネント内にハードコードされたプレイスホルダデータをレンダリングするだけです。
Relayを使用して因位のデータを取得するようにそれを修正しましょう。

`Newsfeed.tsx`ファイルを開いてください。
チュートリアル内のすべてのコンポーネントは`src/component`にあります。
その中で、データがはハードコードされた`<Newsfeed>`コンポーネントを確認できるはずです。

```ts
export default function Newsfeed() {
  const story = {
    title: "Placeholder Story",
    summary:
      "Placeholder data, to be replaced with data fetched via GraphQL",
    poster: {
      name: "Placeholder Person",
      profilePicture: {
        url: "/assets/cat_avatar.png",
      },
    },
    thumbnail: {
      url: "/assets/placeholder.jpeg",
    },
  };
  return (
    <div className="newsfeed">
      <Story story={story} />
    </div>
  );
}
```

このプレイスホルダデータをサーバからデータを取得するように置き換えます。
まず、GraphQLクエリを定義する必要があります。
`Newsfeed`コンポーネントの上に次の宣言を追加してください。

```ts
import { graphql } from 'relay-runtime';

const NewsfeedQuery = graphql`
  query NewsfeedQuery {
    topStory {
      title
      summary
      poster {
        name
        profilePicture {
          url
        }
      }
      thumbnail {
        url
      }
    }
  }
`;
```

これを詳しく確認しましょう。

- JavaScript内にGraphQLを埋め込むために、``` graphql`` ```タグでマークした文字列リテラルを置きます。
  このタグは、Relayコンパイラが　JavaScriptコードベース内にあるGraphQLを探してコンパイルさせます。
- GraphQL文字列は、`query`キーワードとクエリ名で構成されます。
- クエリ宣言の内部は*フィールド*で、何をクエリするか記述します。
  - いくつかのフィールドは文字列、数値または他の単一な情報を取得する*スカラーフィールド*です。
  - 他のフィールドは、1つのノードからグラフ内の他に登壇する*エッジ*です。
    フィールドがエッジのとき、それはエッジのもう一方の端にあるノードのフィールドが含まれる他のブロック`{}`が続きます。
    ここで、`poster`フィールドは`Story`からそれを投稿した`Person`に向かうエッジです。
    一度、`Person`を横断したら、彼らの`name`のような`Person`に関するフィールドを含めます。

これは、このクエリが質問したグラフの一部であることを説明しています。

![query a part of graph](https://relay.dev/assets/images/query-breakdown-56a29935576fa45104147bef7da35749.png)

現在、クエリを定義しましたため、2つのことをする必要があります。

1. Relayが新しいGraphQLクエリを知るために、`npm run relay`でRelayコンパイラを実行します。
2. それを取得して、サーバーによって返されたデータを使用するために、Reactコンポーネントを修正します。

`package.json`を開くと、Relayコンパイラの起動をフックする`relay`スクリプトを確認できます。
これは、`npm run relay`をします。
一度、コンパイラが新しくコンパイルしたクエリを正常に更新または生成すると、`src/components/`の下に生成された**generated**フォルダの中に`NewsfeedQuery.graphql.ts`として見つけることができます。
このプロジェクトは事前にコンパイルされたフラグメントで成り立っているため、このステップをする必要はなく、望んた結果を得られないかもしれません。

> ```sh
> cd relay-examples/newsfeed
> npm run relay
> ```
>
> 上記を実行すると、`*/newsfeed/src/components/__generated__`フォルダに`NewsfeedQuery.graphql.ts`ファイルが生成される。

次に、`Newsfeed`コンポーネントに戻り、プレイスホルダデータの削除から始めます。
その後、次でそれを置き換えます。

```ts
import { useLazyLoadQuery } from "react-relay";

export default function Newsfeed({}) {
  const data = useLazyLoadQuery(
    NewsfeedQuery,
    {},
  );
  const story = data.topStory;
  // As before:
  return (
    <div className="newsfeed">
      <Story story={story} />
    </div>
  );
}
```

`useLazyLoadQuery`フックは、データを取得して返します。
それは2つの引数を受け取ります。

- 前に定義したGraphQLクエリ
- クエリといっしょにサーバーに渡す変数です。高尾クエリは変数を宣言していないため、空のオブジェクトです。

`useLazyLoadQuery`が返すオブジェクトは、そのクエリと同じ形状を持ちます。
例えば、それをJSONフォーマットで出力した場合、次のようになります。

```json
{
  "topStory": {
    "title": "Local Yak Named Yak of the Year",
    "summary": "The annual Yak of the Year awards ceremony ...",
    "poster": {
      "name": "Baller Bovine Board",
      "profilePicture": {
        "url": "/images/baller_bovine_board.jpg",
      },
    },
    "thumbnail": {
      "url": "/images/max_the_yak.jpg",
    }
  }
```

GraphQLクエリによって選択されたそれぞれのフィールドが、JSONレスポンスの属性に対応していることに注意してください。

この時点で、サーバーから取得したストーリを確認できるはずです。

![newsfeed with real data](https://relay.dev/assets/images/queries-basic-screenshot-cdac7c0e384df7a0dbddaf1e3d3f3de2.png)

> **ℹ 注意事項**
> サーバーのレスポンスは、近くできるローディング状態にするために、人為的に遅くしており、それはアプリにより双方向性を追加するときに役に立ちます。
> もし、遅延を削除したい場合、`server/index.js`を開き、`sleep()`呼び出しを削除してください。

> `server/index.mjs`の`sleep()`関数を削除することで、遅延を削除できる。

`useLazyLoadQuery`フックは、コンポーネントが最初にレンダリングされるときにデータを取得します。
また、Relayは、アプリがロードされた後でデータを事前取得するAPIも持っています。
これらについては後で説明します。
いかなる場合でも、Relayはデータが利用可能になるまでローディングインジケーターを表示するために`Suspense`を使用します。

これがRelayの最も基本的な形式です。
コンポーネントがレンダリングされたとき、GraphQLクエリの結果を取得します。
チュートリアルが進むにつれて、Relayの機能がどのように連携して、アプリをより保守性を高くするかを確認できます。
Relayが、それぞれのクエリに対応するTypeScriptの型を生成する方法を確認することから始めましょう。

## Deep Dive: データロードのためのSuspense

`Suspense`はReactの新しい機能で、それはデータを読み込まれる間、Reactがデータを必要とするコンポーネントをレンダリングする前に、待機させます。
レンダリングする前にコンポーネントがデータを読み込む必要があるとき、Reactは読み込み中を示すインジケーターを表示します。
`Suspense`と呼ばれる特別なコンポーネントを使用することで、読み込みインジケーターの位置やスタイルを制御できます。

現時点では、`App.tsx`内部に`Suspense`コンポーネントがあり、それは`useLazyLoadQuery`がデータを読み込んでいる間、スピナーを表示します。

アプリをより双方向性を追加するとき、後のセクションで詳細に`Suspense`を確認します。

## Deep Dive: クエリは静的

RelayアプリのすべてのGraphQL文字列は、Relayコンパイラによって事前に処理され、最終的なバンドルコードから削除されます。
これは、ランタイムでGraphQLクエリを構築できないことを意味します。
それらは静的な文字列リテラルでなくてはならないため、それらはコンパイル時に知られます。
しかし、そこに大きな利点があります。

最初に、それはRelayがクエリの結果の型定義を生成させせることで、コードをより型安全にします。

2つ目に、RelayはGraphQL文字列リテラルを、何をしているかをRelayに伝えるオブジェクトに置き換えます。
これは、ランタイムでGraphQL文字列を直接使用するよりも、より高速です。

また、Relayのコンパイラは、[サーバーにクエリを保存](https://relay.dev/docs/guides/persisted-queries/)するように構成できるため、アプリを構築するとき、ランタイム時にクライアントはクエリ自身の代わりにクエリのIDのみを送信する必要があります。
これは、アプリがバンドルサイズとネットワークのバンド幅を節約して、アプリが構築されたクエリのみが利用可能になるため、攻撃者が悪意のあるクエリを記述することを防げます。

よって、プログラム内にGraphQLでタグ付けされた文字列リテラルを持つとき・・・

```ts
const MyQuery = graphql`
  query MyQuery {
    viewer {
      name
    }
  }
`;
```

・・・JavaScript変数`MyQuery`は、実際に次のようなオブジェクトに割り当てられます。

```ts
const MyQuery = {
  kind: "query",
  selections: [
    {
      name: "viewer",
      kind: "LinkedField",
      selections: [
        name: "name",
        kind: "ScalarField",
      ],
    }
  ]
};
```

> 上記は誤りで、実際は次だと考えられる。
>
> ```ts
> const MyQuery = {
>   kind: "query",
>   selections: [
>     {
>       name: "viewer",
>       kind: "LinkedField",
>       selections: [
>         {
>           name: "name",
>           kind: "ScalarField",
>         }
>       ],
>     }
>   ]
> };
> ```

様々な他のプロパティと情報とともに。
これらのデータ構造は、Relayのペイロード処理コードをとても高速にジックおするためにJITされるために注意深く設計されています。
もし、興味がある場合は、それと一緒に実行するために、[Relay Compiler Explorer](https://relay.dev/compiler-explorer/)を使用できます。

```text
along with various other properties and information. These data structures are carefully designed to allow the JIT to run Relay’s payload processing code very quickly. If you’re curious, you can use the Relay Compiler Explorer to play with it.
```

---

## Relayと型システム

TypeScriptが記述したコードにエラーを報告していることに気付いたかもしれません。

```ts
const story = data.topStory;
                   ^^^^^^^^
Property 'topStory' does not exist on type 'unknown'
```

これを修正するために、Relayが生成した型で`useLazyLoadQuery`の呼び出しを注釈する必要があります。
それによって、TypeScriptは、クエリで選択されたそのフィールドに基づくべきデータの型が何かを知ります。
次を追加してください。

```ts
import type {NewsfeedQuery as NewsfeedQueryType} from './__generated__/NewsfeedQuery.graphql';

function Newsfeed({}) {
  const data = useLazyLoadQuery
  <NewsfeedQueryType>
  (NewsfeedQuery, {});
  ...
}
```

`__generated__/NewsfeedQuery.graphql`の中を確認した場合、次の型定義を確認します。
ちょうど追加した注釈で、TypeScriptは、`data`がこの型であるべきであることを知ります。

```ts
export type NewsfeedQuery$data = {
  readonly topStory: {
    readonly poster: {
      readonly name: string | null;
      readonly profilePicture: {
        readonly url: string;
      } | null;
    };
    readonly summary: string | null;
    readonly thumbnail: {
      readonly url: string;
    } | null;
    readonly title: string;
  } | null;
};
```

Relayコンパイラは、アプリで``` graphql`` ```リテラルを持つすべてのGraphQの部分に対応するTypeScriptの方を生成します。
`npm run dev`を起動している限り、Relayコンパイラは、JavaScriptソースファイルを保存したときはいつでも、これらのファイルを自動的に再生成するため、それらを最新に維持するためにリフレッシュする必要はありません。

Relayの生成した方を使用することは、アプリを安全かつより保守性を高くします。
TypeScriptに加えて、代わりにFlowの型システムを使用したい場合、RelayはFlowの型システムをサポートします。
Flowを使用しているとき、Flowは``` graphql`` ```でタグ付けられたリテラルを直接理解するため、`useLazyLoadQuery`の追加の注釈は必要ありません。

このチュートリアルを通じて型を再訪問しました。
しかし次は、Relayが保守性を向上させる、さらに重要な方法を確認します。

---

## まとめ

クエリはGraphQLデータの取得の基盤です。
次を確認しました。

- ``` graphql`` ```でタグ付けしたリテラルを使用して、アプリ内にGraphQLクエリを定義する方法
- コンポーネントがレンダリングされるときにクエリの結果を取得するために`useLazyLoadQuery`を使用する方法
- 型安全性のためにRelayが生成した型をインポートする方法

次のセクションでは、Relayの最も核となる1つで、特徴的な側面を持つ`Fragment`を確認します。
`Fragment`は、それぞれ個々のコンポーネントがそれ独自のデータ要求を定義する一方で、サーバーに単一クエリを発行する性能面の利点を保持します。
