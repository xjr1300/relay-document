# 型放出(Type Emission)

<https://relay.dev/docs/guides/type-emission/>

- [型放出(Type Emission)](#型放出type-emission)
  - [変数の操作](#変数の操作)
  - [操作とフラグメントのデータ](#操作とフラグメントのデータ)
  - [フラグメント参照](#フラグメント参照)
  - [単独の生成物ディレクトリ](#単独の生成物ディレクトリ)
  - [背景情報](#背景情報)

通常の仕事の一部として、[Relayコンパイラー](https://relay.dev/docs/guides/compiler/)は、型安全なアプリケーションコードの記述を支援する、あなたが選択した言語の型情報を放出します。
これらの型は、`relay-compiler`が操作とフラグメントを記述するために生成した生成物(artifact)内に含まれます。

## 変数の操作

クエリ、ミューテーションまたはサブスクリプション操作のために使用される変数オブジェクトの形状です。

> The shape of the variables object used for query, mutation, or subscription operations.

次の例において、放出された型情報は、非null文字列を使用した`artistID`キーを含むために変数オブジェクトを要求します。

```ts
/**
 * export type ExampleQuery$variables = {
 *   readonly artistID: string
 * }
 * export type ExampleQuery$data = {
 *   readonly artist?: {
 *     readonly name?: string
 *   }
 * }
 * export type ExampleQuery = {
 *   readonly variables: ExampleQuery$variables
 *   readonly response: ExampleQuery$data
 * }
 */
const data = useLazyLoadQuery(
  graphql`
    query ExampleQuery($artistID: ID!) {
      artist(id: $artistID) {
        name
      }
    }
  `,
  // 変数はExampleQuery$variables型であることが期待されます。
  {artistID: 'banksy'},
);
```

## 操作とフラグメントのデータ

操作またはフラグメントで選択されたデータの形状は、[データマスキング規則](https://relay.dev/docs/principles-and-architecture/thinking-in-relay/#data-masking)に従います。
つまり、フラグメントの展開により、任意の選択されたデータが除外されます。

次の例において、放出された型情報は、`useLazyLoadQuery`または`usePreloadedQuery`によって返されたレスポンスデータを説明します。

```ts
/**
 * export type ExampleQuery$variables = {
 *   readonly artistID: string
 * }
 * export type ExampleQuery$data = {
 *   readonly artist?: {
 *     readonly name?: string
 *   }
 * }
 * export type ExampleQuery = {
 *   readonly variables: ExampleQuery$variables
 *   readonly response: ExampleQuery$data
 * }
 */

// dataはExampleQuery$data型です。
const data = useLazyLoadQuery(
  graphql`
    query ExampleQuery($artistID: ID!) {
      artist(id: $artistID) {
        name
      }
    }
  `,
  {artistID: 'banksy'},
);

return props.artist && <div>{props.artist.name} is great!</div>
// 上記は誤りで、正しくは次ではないか？
// return data.artist && <div>{data.artist.name} is great!</div>
```

同様に、次の例において、放出された型情報は、`useFragment`が受け取るために期待するフラグメントの参照の型にマッチするプロップの型を説明します。

```ts
/**
 * export type ExampleFragmentComponent_artist$data = {
 *   readonly name: string
 * }
 *
 * export type ExampleFragmentComponent_artist$key = { ... }
 */

import { ExampleFragmentComponent_artist$key } from "__generated__/ExampleFragmentComponent_artist.graphql"

interface Props {
  artist: ExampleFragmentComponent_artist$key,
};

export default ExampleFragmentComponent(props: Props) {
  // dataはExampleFragmentComponent_artist$data型です。
  const data = useFragment(
    graphql`
      fragment ExampleFragmentComponent_artist on Artist {
        biography
      }
    `,
    props.artist,
  );

  return <div>About the artist: {props.artist.biography}</div>;
  // 上記ではなく、次を説明したかったのでは？
  // return <div>Name of the artist: {data.name}</div>;
}
```

## フラグメント参照

不透明な識別子は、子コンテナがその親から受け取ることを期待してる[データマスキング](https://relay.dev/docs/principles-and-architecture/thinking-in-relay/#data-masking)を記述しています。
それは、子コンテナのフラグメントが、親のフラグメントの内部に展開することを表現します。

> Relayは、子コンポーネントが必要とするデータを示す**フラグメント**を子コンポーネントと同じ場所に記述することで、子コンポーネントで使用するデータの形を保証する。
> ページとなる親コンポーネントは、親コンポーネントが問い合わせるクエリに集約するとともに、子コンポーネントで定義されたフラグメントを参照及び展開して、1回のリクエストでページを連等リングするために必要となるデータを取得する。
>
> フラグメントとコンポーネントを同じファイルに実装する場合、親コンポーネントは子コンポーネントが利用するデータ（フラグメント）の内容を知る必要はない。
> ただし、親コンポーネントは子コンポーネントが使用するデータにアクセスすることは可能で、もしアクセスした場合、子コンポーネントで隠蔽したデータに親コンポーネントが依存することになる。
> 子コンポーネントのデータに親が依存すると、子コンポーネントの変更が親コンポーネントを壊す可能性がある。
>
> 親コンポーネントが子コンポーネントに依存しないように、Relayはデータマスキングと呼ばれる機能で、親コンポーネントが子コンポーネントが隠蔽したデータの可視性を制御する。
>
> 親コンポーネントが子コンポーネントで定義したフラグメントと同じデータを必要とする場合、親コンポーネントでも子コンポーネントと同じフィールドを重複してクエリに追加する。
> Relayは、クエリ内に重複するフィールドは最適化によりマージするため、レスポンスが最適化されるため、上記に問題はない。

> **💡 INFO**
> 実際に型安全なフラグメント参照のチェックを有効にすることについては、[この重要な注意事項](https://relay.dev/docs/guides/type-emission/#single-artifact-directory)を読んでください。

上記フラグメントコンポーネント例で構成されるコンポーネントを考えてください。
この例において、子コンポーネントで放出された型情報は、フラグメント参照と呼ばれる一意で不透明な識別子型を受け取ります。
これは、親のフラグメントのために放出された型情報で、子フラグメントが展開される場所を参照します。
したがって、子フラグメントが親フラグメント内に展開されることを保証して、正確なフラグメント参照は、ランタイムで子コンポーネントに渡されます。

```ts
import { ExampleFragmentComponent } from "./ExampleFragmentComponent"

/**
 * import { ExampleFragmentComponent_artist$fragmentType } from "ExampleFragmentComponent_artist.graphql";
 *
 * export type ExampleQuery$data = {
 *   readonly artist?: {
 *     readonly name: ?string,
 *     readonly " $fragmentSpreads": ExampleFragmentComponent_artist$fragmentType
 *   }
 * }
 * export type ExampleQuery$variables = {
 *   readonly artistID: string
 * }
 * export type ExampleQuery = {
 *   readonly variables: ExampleQuery$variables
 *   readonly response: ExampleQuery$data
 * }
 */

// dataの型はExampleQuery$dataです。
const data = useLazyLoadQuery(
  graphql`
    query ExampleQuery($artistID: ID!) {
      artist(id: $artistID) {
        name
        ...ExampleFragmentComponent_artist
      }
    }
  `,
  {artistID: 'banksy'},
);

// Here only `data.artist.name` is directly visible,
// the marker prop $fragmentSpreads indicates that `data.artist`
// can be used for the component expecting this fragment spread.
// ここで、`data.artist.name`のみが直接見えて、
// マーカープロパティ$fragmentSpreadsは、`data.artist`がこのフラグメント
// の展開を期待するコンポーネントに使用されることを意味します。
return <ExampleFragmentComponent artist={data.artist} />;
```

## 単独の生成物ディレクトリ

注目しなくてはならない重要な注意事項は、デフォルトで厳密なフラグメント参照の型情報は放出されず、代わりにそれらは`any`として型付けされ、子コンポーネントに任意のデータを渡せるようにします。

この機能を有効にするために、コンパイラ設定内に`artifactDirectory`を指定することで、単独のディレクトリにすべての生成物を蓄積することをコンパイラに伝える必要があります。

```json
// package.json
{
  "relay": {
    "artifactDirectory": "./src/__generated__",
    ...
  },
  ...
}
```

そして、さらに`.babelrc`設定ファイル内で、babelプラグインに成果物を探す場所を伝える必要があります。

```text
# .babelrc
{
  "plugins": [
    ["relay", { "artifactDirectory": "./src/__generated__" }]
  ]
}
```

ソースファイルに相対パスを記述しなくても良いように、モジュール解決設定内に、このディレクトリをエイリアスすることが推奨されます。
これは、上記の例でも実施したことで、生成物は`../../../../__generated__`のような相対パスではなく、`__generated__`エイリアスからインポートされます。

> クエリなどが定義したファイルと同じディレクトリの`__generated__`ディレクトリ内に生成されるため、エイリアスを使用すること。

## 背景情報

`relay-compiler`と生成物の放出は状態を持ちません。
これは、オリジナルのソースファイルの場所と、前にコンパイラが付随する生成物を保存したディスク上の場所を追跡しないことを意味します。
したがって、コンパイラーは、*他の*生成物からフラグメント参照の型をインポートすることを試みる生成物を放出する場合、コンパイラは次を行います。

- 最初に、他の生成物が存在するディスク上の場所を知る必要があります。
- そして、他の生成物がディスク上の場所を変更したとき、インポートを更新します。

Facebookは、すべてのソースファイルが平らな名前空間に存在する🤔、[Haste](https://twitter.com/dan_abramov/status/758655309212704768)と呼ばれるモジュールシステムを利用しています。
これは、インポート宣言が他のモジュールへのパスを記述する必要をなくして、コンパイラに上記問題を考える必要をなくすことを意味します。
例えば、インポートでは、モジュールファイル名のベース名を指定することのみを必要にして、Hasteはインポート時に実際に正しいモジュールを検索するために注意を払います。
Facebookの外部で、Hasteモジュールシステムの利用は実績がなく推奨もされていないため、フラグメント参照をインポートせず、代わりに`any`としてそれらを型付けすることを決定しました。

最も単純に言えば、すべてのモジュールファイルを含む単一なディレクトリとしてHasteを捉えることができ、したがって、相対的な兄弟パスを使用した全てのモジュールインポートは常に安全になります。
これは、単一の生成物ディレクトリ機能によって得られるものです。
ソースファイルと同じ場所に配置するよりも、すべての生成物を単一ディレクトリに蓄積することで、コンパイラがフラグメント参照型のインポートを放出できるようにします。
