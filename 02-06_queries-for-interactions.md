# 対話的なクエリ

<https://relay.dev/docs/tutorial/queries-2/>

- [対話的なクエリ](#対話的なクエリ)
  - [クエリ変数](#クエリ変数)
  - [ステップ1 - クエリ変数を定義する](#ステップ1---クエリ変数を定義する)
  - [ステップ2 - フィールド引数として変数を渡す](#ステップ2---フィールド引数として変数を渡す)
  - [ステップ3 - useLazyLoadQueryに引数の値を提供する](#ステップ3---uselazyloadqueryに引数の値を提供する)
  - [ステップ4 - 親コンポーネントからIDを渡す](#ステップ4---親コンポーネントからidを渡す)
    - [Deep Dive: キャッシングとRelayストア](#deep-dive-キャッシングとrelayストア)
    - [Deep Dive: GraphQLが変数構文を必要とする理由](#deep-dive-graphqlが変数構文を必要とする理由)
  - [事前に読み込まれたクエリ](#事前に読み込まれたクエリ)
    - [ステップ1 - useLazyLoadQueryをusePreloadedQueryに変更する](#ステップ1---uselazyloadqueryをusepreloadedqueryに変更する)
    - [ステップ2 - 親コンポーネントからアクセスするためにクエリをエクスポートする](#ステップ2---親コンポーネントからアクセスするためにクエリをエクスポートする)
    - [ステップ3 - 親コンポーネントでuseQueryLoaderを呼び出す](#ステップ3---親コンポーネントでusequeryloaderを呼び出す)
    - [ステップ4 - イベントハンドラー内でクエリを取得する](#ステップ4---イベントハンドラー内でクエリを取得する)
  - [まとめ](#まとめ)

フラグメントがそれぞれのコンポーネント内にデータ要求を指定する方法を確認しましたが、ランタイムで画面全体のために単一のクエリを実行する方法は確認していません。
ここで、同じ画面で2つ目のクエリが欲しい状況を見てみます。
これは、GraphQLクエリのいくつかの機能をより探求することにもなります。

- ストーリーのポスターについて、それらをホバーしたときに、より詳細を表示する**ホバーカード**を構築します。
- ホバーカードは、ホバーした場合にのみ必要な**追加情報**を取得する2つ目のクエリを実行します。
- **クエリ変数**を使用して、より詳細に知りたい人をサーバーに伝えます。
- **事前クエリ**を使用して、性能を改善する方法を確認します。

これらのトピックをカバーした後、フラグメントをより応用した機能をいくつか確認するために戻ります。

---

このセクションにおいて、ストーリーやポスターの名前をホバーすることで、ストーリーのポスターについてより詳細を確認できるように、`PosterByline`にホバーカードを追加します。

> **Deep Dive: 2つ目のクエリを使用するとき**
> 以前、Relayは、画面全体のためにすべてのデータ要求を事前に取得することを支援するように設計されていることを言及しました。
> しかし、これを一般化でき、最大で1つのクエリで持つべき*ユーザーとの双方向性*であると言いました。
> 他の画面に遷移することは、一般的なユーザー操作の1つに過ぎません。
>
> 画面内で、いくつかの操作は、初期時に表示されていたモノから追加データから追加のデータが開示するかもしれません。
> 操作が比較的まれにしか実行されないが、大量の追加データが必要な場合、画面が最初に読み込まれたときに事前に取得するのではなく、操作が発生自他q時に実行される2番目のクエリで、その追加データを取得するほうが賢明です。
> これは、初期読み込みを早くして、より安価にします。
>
> また、例えば、ホバーカードの中のホバーカードなど、データを取得する量を定義できない操作がいくつかあり、静的に知ることは実現不可能です。
>
> もし、データが低い優先度で、メインデータが読み込まれた後で読み込まれなければならないが、それ以上ユーザーの入力なしで、自動でポップアップされなくてはならない場合、そのためにRelayは*遅延フラグメント*と呼ばれる機能があります。
> それは後で説明します。

使用できるホバーカードコンポーネントはすでに準備されています。
しかし、それは`ImageFragment`を使用するためコンパイルエラーを避けるために`future`と呼ばれるディレクトリ内にあります。
現在、`future`にあるモジュールを`src/components`に移動することができる、チュートリアルの段階にいます。

```sh
mv future/* src/components
```

これにより、フラグメントを使用する`PosterByline`を作成するために演習する場合、`PosterByline`コンポーネントは次のように見えるはずです。

```tsx
export default function PosterByline({ poster }: Props): React.ReactElement {
  const data = useFragment(PosterBylineFragment, poster);
  // ref={hoverRef}を追加
  return (
    <div className="byline">
      <Image image={data.profilePicture} width={60} height={60} className="byline__image" />
      <div className="byline__name" ref={hoverRef}>{data.name}</div>
    </div>
  );
}
```

ホバーカードコンポーネントを使用するために、次の変更をしてください。

```tsx
import Hovercard from './Hovercard';
import PosterDetailsHovercardContents from './PosterDetailsHovercardContents';
const {useRef} = React;

...

export default function PosterByline({ poster }: Props): React.ReactElement {
  const data = useFragment(PosterBylineFragment, poster);
  const hoverRef = useRef(null);
  return (
    <div
      ref={hoverRef}
      className="byline">
      <Image image={data.profilePicture} width={60} height={60} className="byline__image" />
      <div className="byline__name">{data.name}</div>
      <Hovercard targetRef={hoverRef}>
        <PosterDetailsHovercardContents />
      </Hovercard>
    </div>
  );
}
```

これにより、誰かの名前をホバーオーバーしたときはいつでも、より情報を持ったホバーカードを表示できるようになるはずです。
`PosterDetailsHovercardContents.tsx`の内部を確認したとき、コンポーネントがマウントされたときに追加情報を取得するために、`useLazyLoadQuery`で2つ目のクエリを実行していることを発見します。

1つ問題があります。
どのポスターをホバーオーバーするか関係なく、常にそれは同じ人の情報を法事します。

---

## クエリ変数

より情報が欲しい情報がどれかをサーバーに伝える必要があります。
GraphQLは特定のフィールドに引数として渡すことができる*クエリ変数*を定義できます。
これらの引数は、サーバーで利用できるようになります。

前のセクションで、フィールドが引数を受け取れるようにする方法を確認しましたが、例えば`url(width: 200, height: 200)`のように、値はハードコードされていました。
クエリ変数を使用して、ランタイムでこれらの値を決定できます。
それらは、それ自身のクエリと一緒にクライアントからサーバーに渡されます。

`PosterDetailsHovercardContents.tsx`の中を確認してください。
次のようなクエリを確認できるはずです。

```tsx
const PosterDetailsHovercardContentsQuery = graphql`
  query PosterDetailsHovercardContentsQuery {
    node(id: "1") {
      ... on Actor {
        ...PosterDetailsHovercardContentsBodyFragment
      }
    }
  }
`;
```

`node`フィールドは、与えられたその一意なIDで任意のグラフノードを取得できるようにするスキーマ内に定義された最上位フィールドです。
`node`フィールドは引数としてIDを受け取り、IDは現在ハードコードされています。
この演習において、このハードコードされたIDを、UIの状態によって供給された変数に置き換えます。

そして、おかしく見える`... on Actor`は*型リファインメント*です。
次のセクションでこれらをより詳細に確認するため、今はそれを無視できます。
簡潔に言えば、`node`フィールドに任意のIDを供給するため、選択するノードの*型*が何か、静的に知る方法はありません。
型リファインメントは、期待する方が何かを指定して、`Actor`型にあるフィールを使用できるようにします。

その中で、表示したいフィールドを含んだフラグメントを単純に拡散します。
それについては後で説明します。
現在のところ、ここはこのハードコードされたIDを、ホバリングオーバーしたポスターのIDで置き換える段階です。

## ステップ1 - クエリ変数を定義する

最初に、クエリ変数を受け取るようにクエリの宣言を編集する必要があります。
ここにその変更を示します。

```tsx
const PosterDetailsHovercardContentsQuery = graphql`
  query PosterDetailsHovercardContentsQuery(
    $posterID: ID!      # ここを変更してクエリの宣言を変更
  ) {
    node(id: "1") {
      ... on Actor {
        ...PosterDetailsHovercardContentsBodyFragment
      }
    }
  }
`;
```

- 変数名は`$posterID`です。
  これはシンボルで、GraphQLクエリの残りで、UIから渡された値を参照します。
- 変数は肩を持ち、この場合は`ID!`です。
  `ID`型は`String`の同義語で、それは他の文字列と区別するためにノードのIDとして仕様されます。
  `ID!`の`!`は非nullを意味します。
  GraphQLにおいて、通常、フィールドはnull許可で、null許可でないのは例外です。

## ステップ2 - フィールド引数として変数を渡す

これにより、ハードコードされた`"1"`を新しい変数に置き換えます。

```tsx
const PosterDetailsHovercardContentsQuery = graphql`
  query PosterDetailsHovercardContentsQuery($posterID: ID!) {
    node(
      id: $posterID     // ここを変更
    ) {
    ... on Actor {
      ...PosterDetailsHovercardContentsBodyFragment
      }
    }
  }
`;
```

> **ℹ 注意事項**
> フィールドの引数としてだけでなく、フラグメントの引数としてもクエリ変数を利用できます。

## ステップ3 - useLazyLoadQueryに引数の値を提供する

これで、ランタイムでUIからの実際の値を渡す必要があります。
`useLazyLoadQuery`フックの2番目の引数は、変数の値を持つオブジェクトです。
コンポーネントに新しいプロップを追加して、その値をここで渡します。

```tsx
export default function PosterDetailsHovercardContents({
  posterID,   // プロップを追加
}: {
  posterID: string;   // プロップの型を定義
}): React.ReactElement {
  const data = useLazyLoadQuery<QueryType>(
    PosterDetailsHovercardContentsQuery,
    {posterID},   // ここで変数を渡す
  );
  return <PosterDetailsHovercardContentsBody poster={data.node} />;
}
```

## ステップ4 - 親コンポーネントからIDを渡す

これで、ホバーカードの親コンポーネントである`PosterByline`から`posterID`プロップを供給する必要があります。
（PosterByline.tsx）ファイルの戦闘に移動して、そのフラグメントに`id`を追加して、プロップとしてIDを渡します。

```tsx
const PosterBylineFragment = graphql`
  fragment PosterBylineFragment on Actor {
    id        # ここを追加
    ...
  }
`;

export default function PosterByline({ poster }: Props): React.ReactElement {
  ...
  return (
   ...
    <PosterDetailsHovercardContents
      posterID={data.id}    // ここでIDを渡す
    />
   ...
  );
}
```

この時点で、ホバーカードは、ホバーオーバーしたそれぞれのポスターのために適切な情報を表示するはずです。

![hovercard](https://relay.dev/assets/images/query-variables-hovercard-correct-4d9278160ae02a07df46a5c395659031.png)

ブラウザのネットワークインスペクターを使用した場合、クエリと一緒に変数値が渡されていることを発見できます。

![pass a variable value](https://relay.dev/assets/images/network-request-with-variables-685cd702f4b24496f003c6dc72a18310.png)

また、このリックエストは、特定のポスターをホバーオーバーした最初のときだけ作成されることに気付いたかもしれません。
Relayはクエリの結果をキャッシュして、その後、それが最近使用されなかった場合にキャッシュされたデータが最終的に削除されるまで、それらを再利用します。

### Deep Dive: キャッシングとRelayストア

他のほとんどのシステムと対照的に、Relayのキャッシングはクエリに基づいていませんが、グラフのノードに基づいています。
Relayは、取得されたすべてのノードのローカルなキャッシュを維持して、それはRelayストアと呼ばれます。
ストア内のそれぞれのノードは識別され、そのIDによって取得されます。

> これが、GraphQLにおいて、ノードのIDが異なるノードを含めて一意でなくてはならない理由である。

もし、2つのクエリが、ノードのIDによって識別される同じ情報を質問した場合、最初のクエリが取得してキャッシュされた情報を使用して満足することで、取得されません。
このキャッシュの振る舞いの利点を得るために[存在しないフィールドのハンドラ](https://relay.dev/docs/guided-tour/reusing-cached-data/filling-in-missing-data/)を確実に設定してください。

Relayは、使用したクエリから"到達可能"でない、またはマウントされたコンポーネントによって最近利用されている場合、ストアからノードをガベージコレクトします。

> Relay will garbage-collect nodes from the Store if they aren’t “reachable” from any queries that are used, or have been recently used, by any mounted components.
>
> haven't been recently used by any mounted componentsの間違いではないか？
> 間違いであれば、上記の文は次の通りに翻訳できる。
>
> Relayは、使用したクエリから"到達可能"でない、またはマウントされたコンポーネントによって最近利用されて**いない**場合、ストアからノードをガベージコレクトします。

### Deep Dive: GraphQLが変数構文を必要とする理由

なぜGraphQLが、変数値をクエリ文字列に補完する代わりに、変数の概念を持っているのか、困惑しているかもしれません。
[前に言及した通り](https://relay.dev/docs/tutorial/queries-1/)、GraphQLクエリ文字列のテキストはランタイムで利用できないため、Relayはそれをより効率的なデータ構造で置き換えます。
Relayは*プリペアードクエリ*を使用するように構成することもできます。
プリペアードクエリでは、ビルド時にコンパイラがそれぞれのクエリをサーバーにアップロードして、IDを割り当てます。
その場合、ランタイムでRelayはサーバーに"クエリ#1337を与えます"と伝えるだけなので、文字列の補完は不可能であり、変数は拘束外になります。
クエリ文字列を利用できるときでも、分離して変数値を渡すことは任意の値をシリアライズして、文字列をエスケープする際に伴う問題が、HTTPリクエストで非通用内情に排除されます。

> Even when the query string is available, passing variable values separately eliminates any issues with serializing arbitrary values and escaping strings, above what is required with any HTTP request.

## 事前に読み込まれたクエリ

このアプリ例はとても単純なため、性能は問題になりません。
実際、サーバーは、読み込み状態を知覚可能にするために、人為的に遅くしています。
しかし、Relayの主要な概念の1つは、現実のアプリの性能を可能な限り高速にすることです。

今すぐ、ホバーカードは`userLazyLoadQuery`を使用して、それはコンポーネントがレンダリングされるときクエリを取得します。
それは、次のようなタイムラインを意味します。

![query timeline](https://relay.dev/assets/images/preloaded-basic-2693ff0ba53d8b3f7e5e905bc6a8d80d.png)

理想的には、可能な限り早くネットワークを経由した取得を開始するべきですが、Reactがレンダリングを終了するまでそれを開始できません。
このタイムラインは、操作が発生したときにホバーカードコンポーネント自身のコードを読み込むために`React.lazy`を使用した場合、さらに悪化する可能性があります。
その場合、次のようになります。

![worse query timeline](https://relay.dev/assets/images/preloaded-lazy-b66d3f27f599118b87b8c917acbed796.png)

GraphQLクエリの取得を開始する前に待機していることに注意してください。
Reactコンポーネントがレンダリングされる前に、マウスイベントハンドラー自体の開始から、クエリの取得が開始されたほうが良いでしょう。
そして、ライムラインは次のようになります。

![starting to fetch query if mouse event handler started](https://relay.dev/assets/images/preloaded-ideal-cd02a128e114e8cbb8dd98f38b1646b1.png)

ユーザーが操作したとき、コンポーネントのレンダリングが開始されている間に、すぐに必要とするクエリの取得を開始するべきです（もし必要であれば最初にそのコードを取得します）。
一度、これら非同期処理が両方とも完了したら、利用できるデータを使用してコンポーネントをレンダリングして、ユーザーにそれを表示できます。

Relayはこれをする*事前クエリ*と呼ばれる機能を提供します。

事前クエリを使用してホバーカードを修正しましょう。

### ステップ1 - useLazyLoadQueryをusePreloadedQueryに変更する

これは、現在データを遅延して取得する`PosterDetailsHovercardContents`であることを思い出してください。

```tsx
export default function PosterDetailsHovercardContents({
  posterID,
}: {
  posterID: string;
}): React.ReactElement {
  const data = useLazyLoadQuery<QueryType>(
    PosterDetailsHovercardContentsQuery,
    {posterID},
  );
  return <PosterDetailsHovercardContentsBody data={data.node} />;
}
```

それは、2番目の引数で*変数*を受け取る`useLazyLoadQuery`を呼び出します。
これを`usePreloadedQuery`に変更します。
しかし、事前クエリを使用する場合、クエリが取得されたときに変数は実際に決定されます。
それは、まだコンポーネントがレンダリングされる前です。
よって、変数の代わりに、このフックはクエリの結果を取得するときにそれが必要とする情報を含む*クエリの参照*を受け取ります。
クエリの参照は、ステップ2でクエリを取得するときに作成されます。

次の通りコンポーネントを変更します。

```tsx
import {usePreloadedQuery} from 'react-relay';
import type {PreloadedQuery} from 'react-relay';
import type {PosterDetailsHovercardContentsQuery as QueryType} from './__generated__/PosterDetailsHovercardContentsQuery.graphql';

export default function PosterDetailsHovercardContents({
  queryRef,
}: {
  queryRef: PreloadedQuery<QueryType>,
}): React.ReactElement {
  const data = usePreloadedQuery(
    PosterDetailsHovercardContentsQuery,
    queryRef,
  );
  ...
}
```

### ステップ2 - 親コンポーネントからアクセスするためにクエリをエクスポートする

親コンポーネントの`PosterByline`が、`PosterDetailsHovercardContentsQuery`クエリを開始するために修正します。
それは、そのクエリへの参照を必要とするため、それをエクスポートする必要があります。

```tsx
export const PosterDetailsHovercardContentsQuery = graphql`...
```

### ステップ3 - 親コンポーネントでuseQueryLoaderを呼び出す

これで、その`PosterDetailsHovercardContents`はクエリ参照を期待するため、クエリ参照を作成して、親コンポーネントである`PosterByline`からそれに渡す必要があります。
`useQueryLoader`を呼び出すフックを使用してクエリ参照を作成します。
また、このフックは、クエリ取得をトリガーするイベントハンドラー内で呼び出す関数を返します。

```tsx
import {useQueryLoader} from 'react-relay';
import type {PosterDetailsHovercardContentsQuery as HovercardQueryType} from './__generated__/PosterDetailsHovercardContentsQuery.graphql';
import {PosterDetailsHovercardContentsQuery} from './PosterDetailsHovercardContents';

export default function PosterByline({ poster }: Props): React.ReactElement {
  ...
  const [
    hovercardQueryRef,
    loadHovercardQuery,
  ] = useQueryLoader<HovercardQueryType>(PosterDetailsHovercardContentsQuery);
  return (
   ...
    <PosterDetailsHovercardContents
      queryRef={hovercardQueryRef}
    />
   ...
  );
}
```

`useQueryLoader`フックは、必要な2つのモノを返します。

- クエリ参照は、`usePreloadedQuery`がクエリの結果を取得するために使用する、不透明な情報のかけらです。
- `loadHovercardQuery`は、リクエストを開始する関数です。

### ステップ4 - イベントハンドラー内でクエリを取得する

最後に、カードが表示されたときに発生するイベントハンドラー内で、`loadHovercardQuery`を呼び出す必要があります。
幸いにも、`Hovercard`コンポーネントは使用する`onBeginHover`イベントがあります。

```tsx
export default function PosterByline({ poster }: Props): React.ReactElement {
  ...
  const [
    hovercardQueryRef,
    loadHovercardQuery,
  ] = useQueryLoader<HovercardQueryType>(PosterDetailsHovercardContentsQuery);
  function onBeginHover() {
    loadHovercardQuery({posterID: data.id});
  }
  return (
    <div className="byline">
      ...
      <Hovercard
        onBeginHover={onBeginHover}
        targetRef={hoverRef}>
        <PosterDetailsHovercardContents queryRef={hovercardQueryRef} />
      </Hovercard>
    </div>
  );
}
```

クエリ変数は、リクエストを開始する場所で渡されることに注意してください。

この時点で、前と同じふるまいを確認できますが、Relayが早くクエリを開始できるため、現在それは少し高速です。

> **💡 TIP**
> 単純に`useLazyLoadQuery`を使用するクエリを導入しても、現実世界で性能を大きく改善できるため、事前読み込まれたクエリは、Relayでクエリを使用する常に好ましい方法です。
> 適切な[サーバーとルーターシステムを統合](https://github.com/relayjs/relay-examples/tree/main/issue-tracker-next-v13)して、クライアントのコードをダウンロードまたは実行する前に、サーバーサイドのWebページのために、メインクエリを事前に読み込むことができます。

## まとめ

- 画面に最初に表示されるすべてのデータは、1つのクエリに結合されるべきですが、将来の情報を必要とするユーザーの操作は2番目のクエリを処理できます。
- クエリ変数は、クエリといっしょにサーバーに情報を渡せるようにします。
- クエリ変数は、フィールド引数にそれらを渡すために使用されます。
- 事前に読み込まれたクエリは常に最善な方法です。
  画面のためにクエリを開始するために、特定のルーティングシステムで可能な限り早く取得を開始してください。
  遅延して読み込むクエリは素早いプロトタイピングのために使用するか、全く使用しないでください。

次に、さまざまなポスターを異なる方法で処理することで、ホバーカードを強化する方法を簡単に確認します。
その後で、最初のクエリの一部になる情報を更新して、別の変数で再取得する必要がある状況を処理する方法を確認します。
