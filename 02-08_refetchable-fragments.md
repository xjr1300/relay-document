# 再取得可能なフラグメント

<https://relay.dev/docs/tutorial/refetchable-fragments/>

- [再取得可能なフラグメント](#再取得可能なフラグメント)
  - [ステップ1 - フラグメント引数を追加する](#ステップ1---フラグメント引数を追加する)
  - [ステップ2 - フィールド引数としてフラグメント引数を渡す](#ステップ2---フィールド引数としてフラグメント引数を渡す)
  - [ステップ3 - @refetchableディレクティブを追加する](#ステップ3---refetchableディレクティブを追加する)
  - [ステップ4 - 検索入力を追加する](#ステップ4---検索入力を追加する)
  - [ステップ5 - useRefetchableFragmentを呼び出す](#ステップ5---userefetchablefragmentを呼び出す)
  - [ステップ6 - useTransitionを使用して読み込みを制御する](#ステップ6---usetransitionを使用して読み込みを制御する)
    - [Deep Dive: どのフラグメントが再取得できるのか？](#deep-dive-どのフラグメントが再取得できるのか)
  - [まとめ](#まとめ)

このセクションでは、ユーザー入力に対するレスポンス内の異なるデータを取得する方法を確認します。

- **フィルタ可能な友人のリスト**を構築します。
- クエリ全体でなく、必要なデータのみを再取得する方法を確認します。

---

Relayは、1つの大きなクエリですべてのデータを取得することを促すため、異なる変数で任意のデータを再取得する必要があったとき、何が発生するでしょうか？

例えば、フィルタ可能なリストを構築しえいることを想定してください。
検索するときに使用する入力値が変更されたとき、新しい検索結果を取得する必要があります。

これをする方法の1つは、リストを取得する分離した2つ目のクエリを使用することで、前にホバーカードで取得するためにしたこととほとんど似ています。
そして、入力が変更されたとき、クエリ変数を変更してクエリを再取得できます。

しかし、任意のユーザー入力が発生する前は、*初期*リストを取得するために2つ目のクエリを使用する必要がないため、これは最適ではありません。
ホバーカードはユーザー操作に応答することでのみ表示されますが、フィルタ可能なリストが表示され、フィルタされる準備ができた場合、それらを大きなクエリの一部のコンテンツとしてその初期コンテンツを得るかもしれません。

一方で、入力が変更されるたびに、*大きなクエリ全体*を再取得支度ありません。
巨大な量のデータを取得することは不必要なだけでなく、UIの他の部分を邪魔する可能性もあることを意味します。
フィルタ可能なリストに関連しないあるデータがサーバー上で変更された場合、クエリが再取得されたとき、データがランダムに変更されたように見えます。
さらに、これはクエリが存在するReactツリーの最上部までユーザー入力をスレッド化することを意味し、十分にスケールしません。

この問題を解決するために、Relayは*再取得可能なフラグメント*を提供します。
これらは、新しい変数で再取得させ、それらが拡散されるクエリの残りから分離するフラグメントです。
ちょうど新しいクエリ変数でクエリ全体を取得できるように、それらはフラグメントの変数を変更して、新しい引数の値で新しいデータを取得できるようにします。

しかし、フラグメントはフラグメントです。
それらはクエリでなく、クエリ内に拡散されることなしで取得されず、クエリ結果を読み出せません。
では、再取得可能なフラグメントは、実際どのように機能するのでしょうか？
回答は、Relayコンパイラは、新しく、フラグメントを取得するクエリを単に生成することです。
データは、最初にフラグメントが拡散されるより大きなクエリの一部として取得されますが、その後、再取得されるときは新しい合成クエリが使用されます。

---

これを試すために、フィルタ可能な連絡先リストを表示するサイドバーをページに追加しましょう。
結局、人に連絡する能力なしで、適切で快適なニュースフィードアプリではないでしょう。

すでに`Sidebar`コンポーネントを準備してあるため、`App.tsx`にそれを単にドロップする必要があります。

```tsx
import Sidebar from './Sidebar';    // 追加

export default function App(): React.ReactElement {
  return (
    <RelayEnvironment>
      <React.Suspense fallback={<LoadingSpinner />}>
        <div className="app">
          <Newsfeed />
          <Sidebar />   {/* 追加 */}
        </div>
      </React.Suspense>
    </RelayEnvironment>
  );
}
```

これで、最上部に人々のリストを表示したサードバーを確認できるはずです。

![sidebar](https://relay.dev/assets/images/refetchable-contacts-initial-6f0b18fcbacc45a5673bbfd8a5a92a76.png)

`ContactsList.tsx`を確認して、連絡先のリストを選択するフラグメントが見つかります。

```tsx
const ContactsListFragment = graphql`
  fragment ContactsListFragment on Viewer {
    contacts {
      id
      ...ContactRowFragment
    }
  }
`;
```

たまたま、`contacts`フィールドはリストをフィルタする`search`引数を受け取ります。
このフラグメント内の`contacts`を`contacts(search: "S")に変更することを試せます。
もし、ページを更新した場合、`S`の文字が含まれる連絡先のみが表示されるはずです。

そして、ゴールは、検索入力を接続して、入力が変更されたときに、`search`引数の新しい値で*このフラグメントだけ*を再取得することです。

> **💡 TIP**
> オプションの演習として、`Sidebar`と`Newsfeed`のクエリを単一のクエリに結合することを試してください。
> `Sidebar`のために、`Newsfeed`と分離したそれ独自のクエリを保つ必要はありません。
> 実際のアプリにおいて、フラグメントと*画面全体*の両方を持つそれらは、単一のクエリのみを持つでしょう。
> チュートリアルにおいて、前の例を単純にするためにクエリを分割してアプリ例を構築しました。

## ステップ1 - フラグメント引数を追加する

最初に、このフラグメントが引数を受け付けるようにする必要があります。
再取得可能なフラグメントを使用して、フラグメント引数は、Relayが生成する取得クエリのクエリ変数になります。
また、それらは通常のフラグメントでも機能して、親クエリは引数に初期値を渡すことができます。

```tsx
const ContactsListFragment = graphql`
  fragment ContactsListFragment on Viewer
    @argumentDefinitions(
      search: {type: "String", defaultValue: null}    # 変数を追加
    )
  {
    contacts {
      id
      ...ContactRowFragment
    }
  }
`;
```

## ステップ2 - フィールド引数としてフラグメント引数を渡す

`contacts`フィールドに引数として、フラグメント引数を渡してください。

```tsx
const ContactsListFragment = graphql`
  fragment ContactsListFragment on Viewer
    @argumentDefinitions(
      search: {type: "String", defaultValue: null}
    )
  {
    contacts(search: $search) {
      id
      ...ContactRowFragment
    }
  }
`;
```

ここの最初の`search`は`contacts`への引数の名前である一方で、2つ目の`$search`はフラグメント引数によって作成された変数であることを、思い出してください。

## ステップ3 - @refetchableディレクティブを追加する

次に、`@refetchable`ディレクティブを追加します。
これは、それを再取得するために追加のクエリを生成することをRelayに伝えます。
生成されたクエリの名前を指定しなくてはなりません。
フラグメントの名前に基づいた名前を付けることを推奨します。

```tsx
const ContactsListFragment = graphql`
  fragment ContactsListFragment on Viewer
    @refetchable(queryName: "ContactsListRefetchQuery")
    @argumentDefinitions(
      search: {type: "String", defaultValue: null}
    )
  {
     // ...
  }
`;
```

## ステップ4 - 検索入力を追加する

ここで、実際にUIとこれを組み合わせる必要があります。
`ContactsList`コンポーネントを確認してください。

```tsx
export default function ContactsList({ viewer }: Props) {
  const data = useFragment(ContactsListFragment, viewer);
  return (
    <Card dim={true}>
      <h3>Contacts</h3>
      {data.contacts.map(contact =>
        <ContactRow key={contact.id} contact={contact} />
      )}
    </Card>
  );
}
```

最初に、検索フィールドを追加する必要があります。

```tsx
import SearchInput from './SearchInput';

const {useState} = React;

function ContactsList({viewer}) {
  const data = useFragment(ContactsListFragment, viewer);
  const [searchString, setSearchString] = useState('');
  const onSearchStringChanged = (value: string) => {
    setSearchString(value);
  };
  return (
    <Card dim={true}>
      <h3>Contacts</h3>
      <SearchInput
        value={searchString}
        onChange={onSearchStringChanged}
      />
      {data.contacts.map(contact =>
        <ContactRow key={contact.id} contact={contact} />
      )}
    </Card>
  );
}
```

## ステップ5 - useRefetchableFragmentを呼び出す

ここで、文字列を変更したときにフラグメントを再取得するために、`useFragment`を`useRefetchableFragment`に変更します。
このフックは、引数として提供した新しい変数を使用してフラグメントを再取得する`refetch`関数を返します。

```tsx
import {useRefetchableFragment} from 'react-relay';

function ContactsList({viewer}) {
  const [data, refetch] = useRefetchableFragment(ContactsListFragment, viewer);
  const [searchString, setSearchString] = useState('');
  const onSearchStringChanged = (value) => {
    setSearchString(value);
    refetch({search: value});
  };
  return (
    // ...
  );
}
```

Relayが、異なる値で再レンダリングするとき、フックへの引数として新しい状態変数を受け付け、再取得するのではなく、再取得するためにコールバックを与えてくれることに気付くでしょう。
これは、イベントが発生するとすぐに取得が開始されることを意味しており、Reactが再レンダリングを完了するまで待機しないため、時間を節約します。
同じ原則は、事前ロードされたクエリで確認しました。
また、それは例えば再取得の跳ね返りを防ぎ（debounce: デバウンス）たい場合に、より制御を与えてくれます。

## ステップ6 - useTransitionを使用して読み込みを制御する

ここで、フラグメントが再取得されたとき、Relayは新しいデータを読み込んでいる間`Suspense`を使用するため、コンポーネント全体がスピナーで置き換えられます。
これは、UIをとても不便にします。
新しいデータが利用できるようになるまで、単純に画面に現在のデータを保持したいでしょう。

`Suspense`は通常次のように機能します。
コンポーネントは、コンポーネントが後で再取得するように、レンダリングするために必要なデータがないとき、`Suspense`はReactに待機するように伝えます。
これが発生したとき、Reactはツリーの最も近い`Suspense`コンポーネントを見つけます。
そして、Reactはそのコンポーネント（`Suspense`）の下のすべてのコンポーネントを"フォールバック"インジケーターで置き換えます。

![when there is missing data](https://relay.dev/assets/images/refetchable-suspense-1-data-needed-05e9379009624a3afdac9e422da834e5.png)

![nearest suspense component](https://relay.dev/assets/images/refetechable-suspense-2-nearest-suspense-point-be5a2c65ed3a31ad3d32e1fb26c8c0b0.png)

![replacing with suspense](https://relay.dev/assets/images/refetchable-suspense-3-fallback-d274cf296d7605ea275d4564cf529adf.png)

これは、画面を初期に読み出すときは理にかないますが、この場合、存在するUIを隠して、それをスピナーで置き換える理由はありません。

これを達成するために、*トランジション*として再取得をマークできます。
トランジションはすぐに応答する必要がない、Reactの状態の更新です。
Reactはデータが利用できるようになるまで待機します。

トランジションは、`useTransition`フックによって提供される関数の呼び出して、状態の変更をラップすることによってマークされます。
これを使用したコードは次のようになります。

```tsx
const {useState, useTransition} = React;

function ContactsList({viewer}) {
  const [isPending, startTransition] = useTransition();
  const [searchString, setSearchString] = useState('');
  const [data, refetch] = useRefetchableFragment(ContactsListFragment, viewer);
  const onSearchStringChanged = (value) => {
    setSearchString(value);
    startTransition(() => {
      refetch({search: value});
    });
  };
  return (
    <Card dim={true}>
      <h3>Contacts</h3>
      <SearchInput
        value={searchString}
        onChange={onSearchStringChanged}
        isPending={isPending}
      />
      {data.contacts.map(contact =>
        <ContactRow key={contact.id} contact={contact} />
      )}
    </Card>
  );
}
```

Reactが新しいデータを待っている間、`Suspense`のフォールバックを使用する代わりに、Reactは`isPending`フラグをtrueに設定してコンポーネントを再レンダリングします。

再取得が発生している間、`isPending`フラグを`SearchInput`に渡しています。
一方、`setSearchString`をトランジションの外側に配置して、`refetch`をその中に配置することで、すぐに検索インプットを更新するようにReactに伝えます。

スピナーを表示しますが、読み込みの間、前のデータの表示を維持することで、よいユーザー体験で連絡先リストを更新することができるようにできました。

![showing the spinner and the previous data](https://relay.dev/assets/images/refetchable-transition-search-b916c9e6ffbb26b6e64c9f1e19fd6fd3.png)

### Deep Dive: どのフラグメントが再取得できるのか？

フラグメントを再取得するために、Relayはフラグメントからの情報を単純に再取得するクエリを生成する方法を知らなくてはなりません。
それは、特定の要件を満たすフラグメントのみ可能です。

少なくとも、フラグメントがその中に拡散するオリジナルのクエリを再実行できると考えているかもしれません。
しかし、GraphQLは、様々なときに、同じクエリが同じ結果を返すことを保証しません。
例えば、サイト全体で最も人気のある投稿を返すGraphQLフィールドがあることを想像してください。

```graphql
query MyQuery {
  topTrendingPosts {
    title
    summary
    date
    poster {
     ...PosterFragment
    }
  }
}
```

単純に、このクエリから`PosterFragment`を再取得したい場合、次のようなクエリを構築して呼び出します。

```graphql
query MyQuery {
  topTrendingPosts {
    poster {
     ...PosterFragment
    }
  }
}
```

最も人気のある投稿は、それを更新したときにより、異なる投稿になる可能性があります。

Relayは、オリジナルのクエリが使用した同じパスによって、もはや到達できなくなった場合でも、フラグメントが最終的に到達するグラフ内の特定のノードを識別する方法を必要とします。
ノードが一意で安定したIDを持っている場合、次のように"特定のIDを持つグラフのノード"とクエリするための規則があります。

```graphql
query RefetchQuery {
  node(id: "abcdef") {
    ...PosterFragment
  }
}
```

実際に、これがRelayが使用する正確な規則です。
Relayは、IDを受け取ってグラフのノードを与える、`node`と呼ばれる最上位フィールドを実装することを期待しています。
ホバーカードの例で、前に`node`を確認しました。
2つ目のクエリを使用して、与えられたIDで特定の人物を取得するために使用されました。

すべてのグラフのノードが安定したIDを持っているわけではなく、いくつかは一時的です。
`node`で使用されるために、スキーマはその型に`Node`と呼ばれるインターフェイスを実装した宣言をしなくてはなりません。

```graphql
type Person implements Node {
  id: ID!
  ...
}
```

単純に言えば、`Node`インターフェイスはIDを持っていることを示していますが、より重要なことは、慣例により、そのIDが安定して一意であることです。

```graphql
interface Node {
  id: ID!
}
```

`Node`を実装する型のフラグメント以外に、ビューアはセッションを通じて安定していると想定されるため、`Viewer`にあるフラグメントや、その上にIDを変更する可能性があるフィールドがないため、最上位にあるフラグメントを再取得することもできます。

## まとめ

再取得可能なフラグメントは、ユーザー入力に応答するUIの特定の部分を効率的に更新する一方で、画面全体で使用する同じクエリの一部としてそれらを初期化します。

また、Relayのページ分け機能は再取得可能なフラグメントの上に構築されています。
それらを次に探求します。
