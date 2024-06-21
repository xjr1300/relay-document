# GraphQLの型、インターフェイス、そしてポリモーフィズム

<https://relay.dev/docs/tutorial/interfaces-polymorphism/>

- [GraphQLの型、インターフェイス、そしてポリモーフィズム](#graphqlの型インターフェイスそしてポリモーフィズム)
  - [まとめ](#まとめ)

このセクションでは、様々な型のノードを異なる方法で扱う方法を確認します。
アプリ例のいくつかのニュースフィードストーリーは人によって投稿される一方で、ほかは組織によって投稿されることに気付いたかもしれません。
この例では、人特有の情報と、組織特有の情報を選択するフラグメントを記述することによって、ホバーカードを強化します。

---

GraphQLのノードは単に、乱雑なバグのフィールドではなく、型を持っています。
GraphQLスキーマは、フィールドそれぞれの型を持っています。
例えば、GraphQLスキーマは次のように`Story`型を定義しているかもしれません。

```graphql
type Story {
  id: ID!
  title: String
  summary: String
  createdAt: Date
  poster: Actor
  image: Image
}
```

個々で、いくつかのフィールドは、`String`と`ID`のようなスカラーです。
`Image`のような他の型は、スキーマの他の場所で定義されています。
これらのフィールドは、それら特定な型のノードへのエッジです。
`ID!`の`!`は、そのフィールドが非nullであることを意味します。
GraphQLにおいて、通常、フィールドはnull許容で、非nullは例外です。

フラグメントは、常に特定の型を"on"しています。
上記例において、`StoryFragment`は`on Story`を定義されています。
これは、`Story`ノードが期待されるクエリ内の場所にのみ展開できることを意味します。
そして、それはフラグメントは`Story`型に存在するそれらのフィールドを単に選択することを意味します。

特に興味深いことは、`poster`フィールドに使用される`Actor`型です。
この型は*インターフェイス*です。
それは、ストーリーの`poster`は`Person`、`Page`、`Organization`、または"Actor"である仕様を満足する任意の他の型のエンティティに成り得ることを意味します。

アプリ例におけるGraphQLスキーマは、次の通り`Actor`を定義しています。

```graphql
interface Actor {
  name: String
  profilePicture: Image
  joined: DateTime
}
```

偶然ではなく、これはここで表示している正確な情報です。
`Actor`を*実装した*スキーマ内の型が2つあり、それが`Actor`で定義されたすべてのフィールドを持つことを意味しており、次のように宣言しています。

```graphql
type Person implements Actor {
  id: ID!
  name: String
  profilePicture: Image
  joined: DateTime
  email: String
  location: Location
}

type Organization implements Actor {
  id: ID!
  name: String
  profilePicture: Image
  joined: DateTime
  organizationKind: OrganizationKind
}
```

これら両方の型は、`name`、`profilePicture`そして`joined`を持っているため、それらは両方とも`Actor`を実装することを宣言しており、それによってスキーマ内やフラグメント内で`Actor`が呼び出される場所で使用できます。
また、それらはそれぞれ特定の型を区別する他のフィールドも持っています。

`Person`の位置、または`Organization`の`OrganizationKind`を表示するために`PosterDetailsHovercardContentsBody`を拡張することで、インターフェイスが機能する方法を確認しましょう。
これらはそれら特定の型のみに存在するフィールドで、`Actor`インターフェイスに存在しません。

これまで従ってきた場合、`PosterDetailsHovercardContents.tsx`内に次のように定義されたフラグメントを持っています。

```graphql
fragment PosterDetailsHovercardContentsBodyFragment on Actor {
  name
  joined
  profilePicture {
    ...ImageFragment
  }
}
```

このフラグメントに`organizationKind`のようなフィールドを追加することを試みた場合、Relayコンパイラから次のエラーを得ます。

```text
✖︎ The type `Actor` has no field organizationKind
```

これは、フラグメントをインターフェイス上にあるものとして定義したためで、そのインターフェイスのフィールドのみを使用できます。
インターフェイスを実装した特定の型のフィールドを使用するために、その型のフィールドを選択することをGraphQLに伝えるために、*型リファインメント*を使用します。

```graphql
fragment PosterDetailsHovercardContentsBodyFragment on Actor {
  name
  joined
  profilePicture {
    ...ImageFragment
  }
  # 以下を追加
  ... on Organization {
    organizationKind
  }
}
```

先に進んで、次を追加してください。
また、`Person`も型リファインメントを追加します。

```graphql
fragment PosterDetailsHovercardContentsBodyFragment on Actor {
  name
  joined
  profilePicture {
    ...ImageFragment
  }
  ... on Organization {
    organizationKind
  }
  # 次を追加
  ... on Person {
    location {
      name
    }
  }
}
```

インターフェイスを実装したいくつかの型にのみ存在するフィールドを選択して、扱っているノードが異なる型であるとき、そのフィールドを読み出すと、単純に`null`が得られます。
それを念頭に置いて、人の場所と組織の組織の種類を表示するために、`PosterDetailsHovercardContentsBody`を修正できます。

```tsx
import OrganizationKind from './OrganizationKind';

function PosterDetailsHovercardContentsBody({ poster }: Props): React.ReactElement {
  const data = useFragment(PosterDetailsHovercardContentsBodyFragment, poster);
  return (
    <>
      <Image image={data.profilePicture} width={128} height={128} className="posterHovercard__image" />
      <div className="posterHovercard__name">{data.name}</div>
      <ul className="posterHovercard__details">
         <li>Joined <Timestamp time={poster.joined} /></li>
         {/* ここから */}
         {data.location && <li>{data.location.name}</li>}
         {data.organizationKind && (
           <li>
             <OrganizationKind kind={data.organizationKind} />
           </li>
         )}
         {/* ここまでを追加 */}
      </ul>
    </>
  );
}
```

これで、人の位置と、組織の組織の種類を確認できるはずです。

![kind of organization](https://relay.dev/assets/images/interfaces-organization-screenshot-3614512165c0726ffd8c8b5e30a8ee6a.png)

![location of people](https://relay.dev/assets/images/interfaces-person-screenshot-4926f665a489443785ec6223110fe725.png)

ところで、現在、前の例で`... on Actor`を持つ理由を理解できました。
ランタイムで任意のIDを与えることができるため、`node`フィールドは任意の型のノードを返すことができます。
したがって、それが与える型は`Node`で、`id`フィールドを持つ任意のモノを実装させられるとても一般的なインターフェイスです。
`Actor`インターフェイスからのフィールドを使用するために、型リファインメントが必要になりました。

> **ℹ 注意事項**
> GraphQL仕様と他の情報源において、型リファインメントは*インラインフラグメント*と呼ばれています。
> 型リファインメントという専門用語はより説明的で混同を少なくするため、それらを"型リファインメント"と読んでいます。

> **💡 TIP**
> 全く異なる型に依存してなにかする必要がある場合、`__typename`と呼ばれるフィールドを選択でき、それは、例えば"Person"または"Organization"のような具体的な型の名前を文字列で返します。
> これは、GraphQLの組み込み機能です。

## まとめ

`... on Type {}`構文は、インターフェイスを実装した特定の型にあるフィールドのみを選択できるようにします。
